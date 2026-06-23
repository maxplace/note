# mallwms-whc-gateway 模块分析说明

  

本文基于 `mallwms-whc-gateway` 工程现有代码进行整理，从模块结构、业务说明、对外接口、底层数据存储、外部接口（HSF）调用、消息（MetaQ / Link）、缓存（Redis）几个维度做一份概览说明。所有函数签名、表名、Topic、tag、key 前缀等均从源码中提取。

  

---

  

## 一、工程结构与模块职责

  

`mallwms-whc-gateway` 是 mallwms 下负责与 4PX（whc / fbx）网关交互的子工程，采用经典分层 Maven 多模块结构：

  

| 模块 | 职责 |

| --- | --- |

| whc-gateway-common | 通用常量、配置开关（SwitchCenter `@AppSwitch`）、工具类（DateUtil 等） |

| whc-gateway-client | 对外暴露的 HSF 接口与 DTO，供其它应用集成 |

| whc-gateway-infra | 基础设施层：HSF 配置、MQ 生产者、MetaQ 消费者基类、Wrapper 封装外部 HSF 服务、日志/异常/工具 |

| whc-gateway-dal | DAO 层：MyBatis Mapper、TDDL 数据源、DO、`*.xml` SQL |

| whc-gateway-biz | 业务核心：策略（receive/send）、解决方案（solution）、Manager、消费者、生产者、定时任务、本地缓存 |

| whc-gateway-service | HSF Provider 实现（`@HSFProvider`），桥接 client 接口与 biz 实现 |

  

入口模式为「网关」：

- 接收 4PX 透传的 `WHC_COMMON_RECEIVE` 报文 → 路由到 receive 策略 → 调用菜鸟 WHC 内部接口落地。

- 监听菜鸟 WHC 的业务消息 (MetaQ) → 路由到 send 策略 → 通过 Link/`WHC_COMMON_SEND` 推送回 4PX。

- 同时对外提供货品同步、清关校验、库调/盘点单创建、运维操作等 HSF 接口。

  

---

  

## 二、业务说明

  

### 1. 出/入库报文接收（receive）

统一入口：`WhcReceiveServiceImpl`（实现 `WhcCommonReceiveService`，使用 `@LinkCallbackProvider(serviceInterface = "WHC_COMMON_RECEIVE")`）。

- 解析 `logistics_interface` XML/JSON，取出 `actionCode` 或 URL 中的 `WHC_RECEIVE_RESTFUL/{method}` 路由 key。

- 通过 `MethodNameConvertManager` 转换为内部 `method`，再由 `ReceiveStrategyFactory` 派发到具体 `AbstractReceiveStrategy`。

  

接收侧策略（method → 业务）：

  

| method | 策略类 | 业务含义 |

| --- | --- | --- |

| `entry.order.create` | `EntryOrderCreate` | 创建入库单（4PX → mallwms） |

| `entry.order.Abnormal.Approval` | `EntryOrderAbnormalDeal` | 入库异常审批/处理 |

| `delivery.order` | `DeliveryorderCreate` | 创建发货/委托单 |

| `outBoundOrderCreate` | `OutboundOrderCreate` | 创建出库单 |

| `orderCancel` | `CancelOrder` | 单据取消 |

| `asyncOrderCancel` | `AsyncCancelOrder` | 异步单据取消（产生延迟消息收尾） |

| `updateMailinfo` | `UpdateConsignOrderMailInfo` | 出库委托单运单/地址变更 |

| `goodsPublish` | `GoodsPublish` | 货品发布同步 |

| `goodsReMeasure` | `GoodsReMeasure` | 货品重新测量 |

| `vasOrderCancel` | `ValueAddCancelOrder` | 增值服务单取消 |

| `4px-create-value-add-order` | `ValueAddCreate` | 增值服务单创建 |

| 动态 msgCode | `ValueAddAuditConfirm` | 增值任务审核结果回告 |

  

策略统一通过 `WhcInterfaceWrapper` 调用菜鸟 WHC 写服务（`WhcOrderWriteService`、`WhcCreateService`、`WhcTradeOrderWriteService`、`ValueAddOrderWriteService` 等）落库。

  

### 2. 业务消息回传（send / msg）

菜鸟内部业务变更通过 MetaQ 通知本应用（`WhcMetaqListener`），按 `messageExt.getTags()` 路由至 `AbstractMsgStrategy`，再走 `SendSolution` 通过 `ConciseLinkClient` 以 `WHC_COMMON_SEND`（`toCode=fpx-gateway-wms-common`，`fromAppKey=mallwms`）回写 4PX。

  

| MetaQ Tag | 策略 | 业务 |

| --- | --- | --- |

| `whc-order-status` | `OrderStatusStrategy` | 单据状态变更 |

| `order-receive-inbound` | `EntryOrderReceiveStrategy` | 入库收货 |

| `order-confirm-inbound` | `EntryOrderPutAwayStrategy` | 入库上架确认 |

| `order-confirm-consign` | `ConsignOrderConfirmConsignStrategy` | 委托单出库装车确认 |

| `order-confirm-outbound` | `ConsignOrderConfirmOutboundStrategy` | 出库确认 |

| `wms-message-upload` | `MessageUploadStrategy` | 通用消息上抛（车辆中转等） |

| `value-add-order-status` | `ValueAddStatusStrategy` | 增值服务单状态 |

| 内部触发 | `WarehouseAbnomalStrategy` | 异常指令上传 (`WhcAbnomalCreateTemplate`) |

  

订阅白名单由 SwitchCenter 配置 `WhcMallWmsConfig.subTagList` 控制；`WhcCommonSendConfig.actionCodeBlackList` 用于发送黑名单。

  

### 3. 货品/库存同步（client + biz/manager）

- `GoodsServiceImpl`：上报货品测量数据 `reportGoodsMeasureData`、从 4PX 拉取货品 `pullGoodsFrom4Px`。

- `GoodsSyncProcessor`（`@JavaProcessor`，SchedulerX 定时任务）：批量从 4PX 拉取货品。

- `InitServiceImpl`：库存/SKU 初始化转换。

  

### 4. 清关校验（CustomsClearance）

- `CustomsClearanceService` 提供出库清关校验、失败回告（单条/批量）。

- 由 `CustomsClearanceVerificationStrategy` / `CustomsClearanceVerificationCallBackStrategy` 实现下发。

- `FpxClearanceVerificationManager` 使用 Redis（`ConcurrentLockSupport`）做身份 ID 唯一性校验。

  

### 5. 仓内单据创建（盘点/库调/不平衡）

`WarehouseCreateOrderService` 暴露 HSF：`createWarehouseCheckOrder`、`createWarehouseAdjustOrder`、`createImbalanceOrder`，分别对应 `WarehouseCheckOrderStrategy`、`WarehouseAdjustOrderStrategy`、`WarehouseImbalanceOrderStrategy`，通过 `WhcOrderWrapper` 落到 WHC。

  

### 6. 运维操作

`MallWmsOpsService.repushFpxMsg` 用于将历史单据按 `actionCode` 重发回 4PX，由 `FpxOrderAssistant`/`FpxOrderConfirmAssistant` 协助构造数据。

  

### 7. 增值任务（VAS）扩展点

`ExtensionRouter` + `IBizSolutionHandler` 形成 SPI 路由：根据 `bizSource` + `match()` 选择具体扩展实现，例如 `FpxOutOrderStatusBizHandlerImpl`、`FpxEntryOrderPutAwayBizHandler` 等，便于按 4PX/默认两套场景隔离。

  

---

  

## 三、对外接口（HSF Provider）说明

  

所有 client jar 中的接口均通过 `@HSFProvider` 暴露，序列化版本号 `${spring.hsf.version}`（部分使用 `${spring.hsf.mallwms.whc.version}`）。

  

### 1. `com.cainiao.whc.gateway.client.service.GoodsService`

- `Result<Void> reportGoodsMeasureData(GoodsMeasureDTO)`：触发货品认证数据回告 4PX。

- `Result<Void> pullGoodsFrom4Px(String ownerCode, List<String> skuCodes)`：按货主、sku 同步 4PX 货品。

  

### 2. `com.cainiao.whc.gateway.client.service.InitService`

- `String pullGoodsFrom4Px(List<String> pullAllOwnerCodeList)`：批量初始化货主货品。

- `List<String> skuConvert(String customerCode, List<String> skuCode)`：4PX → mallwms SKU 转换。

- `List<String> stockConvert(List<String> stockStr)`：库存初始化转换。

  

### 3. `com.cainiao.whc.gateway.client.service.CustomsClearanceService`

- `Result<Boolean> verification(VerificationPackageParam)`：出库清关校验。

- `Result<Boolean> verificationCallBack(VerificationPackageParam)`：清关校验失败回告。

- `Result<List<Boolean>> verification(List<VerificationPackageParam>)`：批量清关校验。

- 静态 method 名：`CustomsClearanceService.verification` / `verificationCallBack`。

  

### 4. `com.cainiao.whc.gateway.client.service.OwnerConfigService`

- `Result<List<String>> getOwnerStoreCodes(String ownerId)`：货主对应仓列表。

- `Result<Void> addOwnerStoreCode(String ownerId, String storeCode)`：新增货主仓关系。

- `Result<String> getOwnerServiceItemId(String ownerId)`：获取货主服务商品 ID。

- `Result<FpxOwnerMappingDTO> getFpxOwnerMappingByOwnerId(String ownerId)`：4PX O-W 货主映射查询。

  

### 5. `com.cainiao.whc.gateway.client.service.MallWmsOpsService`

- `void repushFpxMsg(List<String> orderCodeList, String actionCode, String suffix)`：单据消息重推。

  

### 6. `com.cainiao.wmp.mallwms.whc.gateway.client.service.WarehouseCreateOrderService`

- `Result createWarehouseCheckOrder(WarehouseCheckOrderDTO)`：创建盘点单。

- `Result createWarehouseAdjustOrder(WarehouseAdjustOrderDTO)`：创建库调单。

- `Result createImbalanceOrder(WarehouseImbalanceDTO)`：创建库存不平衡单。

  

### 7. `com.cainiao.wmp.mallwms.whc.gateway.client.service.WhcGatewaySendService`

通用 WHC 网关发送出口，便于上游直接拼好 method+body：

- `String sendWhcGateway(String body)`

- `String invoke(String method, String body)`

- `String sendWhcGateway2(SendMessage sendMessage)`

- `String sendSynWhcGateway(String outMsgId, String body)`

- `String sendSynWhcGatewayByLinkCLinetV2(String outMsgId, String body)`

  

### 8. Jagger / Link 服务

- `WhcCommonSendTemplateImpl`（`@JaggerService(serviceInterface = WhcCommonSendTemplate.class)`）：暴露给 whc 内部走 Jagger 通用发送。

- `WhcAbnomalCreateTemplateImpl`（`@JaggerService(serviceInterface = WhcAbnomalCreateTemplate.class)`）：异常指令创建。

- `WhcReceiveServiceImpl`：实现 `WhcCommonReceiveService`，由 PAC/Link 框架回调。

  

---

  

## 四、底层数据存储

  

模块独立连接 `cainiao_whc` 库（由 `WhcGatewayDataSourceConfig` 通过 TDDL `mallwms.whc.tddl.appname` 配置接入），与 mallwms 默认数据源隔离：

- DataSource Bean：`whcGatewayDatasource`

- SqlSessionFactory：`whcGatewaySqlSessionFactory`

- MapperScan：`com.cainiao.wmp.mallwms.whc.gateway.dal.mapper`，注解 `@WhcDBMapper`

- Mapper xml：`classpath*:/mapper/whc/*.xml`

  

### 表清单

  

#### 1. `fpx_owner_mapping`（4PX O-W 货主映射）

DO：`FpxOwnerMappingDO`，Mapper：`FpxOwnerMappingMapper` (`sqlmap_FpxOwnerMapping.xml`)。

  

| 字段 | 类型 | 含义 |

| --- | --- | --- |

| id | bigint PK | 主键 |

| gmt_create / gmt_modified | datetime | 时间戳 |

| customer_id | varchar | 4PX 客户ID（`MallWmsSyncLinkDTO.customerId`）|

| customer_code | varchar | 助记码 (primaryCustomerCode) |

| wms_owner_id | bigint | 菜鸟货主id (cnitem ownerId) |

  

操作：`insert` / `batchInsert` / `deleteById` / `updateById` / `selectById` / `selectByCustomer(customerId, customerCode)` / `selectByWmsOwnerId` / `selectByWmsOwnerIds` / `selectByCustomers`。

  

#### 2. `fpx_owner_warehouse_relation`（货主-仓关系）

DO：`FpxOwnerWarehouseRelationDO`，Mapper：`FpxOwnerWarehouseRelationMapper`。

  

| 字段 | 类型 | 含义 |

| --- | --- | --- |

| id | bigint PK | 主键 |

| gmt_create / gmt_modified | datetime | 时间戳 |

| wms_owner_id | bigint | 菜鸟货主 id |

| warehouse_code | varchar | 仓库 code |

  

操作：`insert` / `batchInsert(recordList)` / `deleteById` / `updateById` / `selectById` / `selectByOwnerAndWarehouse` / `selectByWmsOwnerId` / `selectByWarehouseCode` / `selectAll` / `selectByWarehouseCodeAndWmsOwnerIdList`。

  

> 工程内未持久化业务流水（入库单/出库单等），单据数据均通过 HSF 写入菜鸟 WHC 系统。

  

---

  

## 五、外部接口调用

  

### 1. HSF Consumer（通过 `@HSFConsumer` 注入）

  

`WhcGatewayHsfConfig`（菜鸟 warehouse / WHC 域）：

- `WhcOrderWriteService`（`1.0.0.inner`，5s）

- `WhcCreateService`（`1.0.0`）

- `WhcTradeOrderWriteService`（`1.0.0`）

- `WhcAbnomalInstructionTemplate`（`1.0.0.whc`）

- `MessageSenderService`（`${spring.hsf.version}`，15s）

- `ValueAddOrderWriteService` / `ValueAddOrderReadService`（`1.0.0`）

- `WhcCommonSendTemplate`（`1.0.0.whc`）

- `WhcExtendService`（`1.0.0`）

  

`GwWhcHsfConfig`：

- `WhcOrderReadService`

  

`GwCnitemHsfConfig`（cnitem 中心）：

- `EnterpriseClientService`、`bpGoodsService(GoodsService)`、`OwnerService`、`ItemBizReceiveService`

  

`GwWmpItemctrHsfConfig`：

- `AuthenticationService`、`ItemMeasureService`

  

`GwCnMemberHsfConfig`：

- 菜鸟会员（`cnmember.hsf.version`）相关服务

  

`GwLinkClientConfig` / `GwLinkConfig`：

- 配置 `ConciseLinkClient` / `LinkClient` 等 Link SDK Bean，用于 `WHC_COMMON_SEND`、`WHC_COMMON_RECEIVE`。

  

### 2. Wrapper 层（统一封装并加日志/异常）

- `WhcInterfaceWrapper`：聚合 WHC 写、创建、Trade、Abnomal、增值、Common Send、Extend、MessageSender 等服务。

- `WhcOrderWrapper`：WHC 订单读 (`wgWhcOrderReadService`)。

- `GoodsWrapper`：cnitem 货品 `bpGoodsService`。

- `OwnerWrapper`：货主与企业 `wgEnterpriseClientService` + `OwnerService`。

- `CnMemberWrapper`：会员中心。

- `DivisionWrapper`：行政区划（`globalDivisionManager`）。

- `SpaceWrapper`：仓内空间。

- `EntruckingWrapper`：装车中转。

- `WgHuWrapper`：HU/容器服务。

- `BatchWrapper`：批次。

  

### 3. 平台协议

- 接收：`com.taobao.pac.client.sdk.receiveservice.WhcCommonReceiveService`（PAC/Link 回调）。

- 发送：`com.cainiao.link.client.ConciseLinkClient`，统一通过 `SendSolution.sendMessage` 走 `WHC_COMMON_SEND`，目标 `toCode = fpx-gateway-wms-common`，`fromAppKey = mallwms`。

  

---

  

## 六、消息（MetaQ / RocketMQ）

  

### 1. 业务消息消费（来自菜鸟 WHC）

- 消费组与 Topic 由 `WhcMetaqListener`（继承 `AbstractMetaqConsumer`）通过 `getTopic` / `getGroup` / `getTagExpression` 提供，按 tag 路由到 `WhcMessageStrategyFactory`。

- 消息体使用 JDK 序列化（`ObjectInputStream`）反序列化为 `WhcOrderBaseDTO`、`WlbPackageDTO` 等 DTO。

- 重投递 ≥15 次时设置 `DelayLevel = 17 (≈1h)` 兜底。

- 已订阅的 tags 由 SwitchCenter `WhcMallWmsConfig.subTagList` 白名单控制，当前默认值：`wms-message-upload`、`whc-order-status`、`order-receive-inbound`、`order-confirm-inbound`、`order-confirm-consign`、`test`。

- 完整 tag → 策略映射见 §二.2。

  

### 2. 自定义延迟消息（应用内）

  

生产者 `DelayMessageProducer`：

- Topic：`mallwms-delay-message`

- ProducerGroup：`MALL_WMS_DELAY_PRODUCER_GROUP`（在 `MetaqProducerConfig` 中初始化 `MetaProducer`）

- 通过 `Message.setDelayTimeLevel(level)` 实现 1s / 5s / 10s / 30s / 1m / … / 2h 多档延迟。

- 主要发送方法：

- `sendFpxChangeMailMq(SynUpdateOtaskAddress)` → tag `FPX_MAIL_CHANGE`，10s 延迟（地址/运单号变更补偿）

- 异步取消后处理 → tag `AYSNC_CANCEL_POST_HANDLE`（`AsyncCancelPostHandleDTO`）

- 出库确认延迟处理 → tag `FPX_OUTBOUND_HANDLE`（`OrderOutConfirmHandleDTO`）

  

消费者 `MetaqConsumerConfig` + `DelayMessageListener`：

- 消费组：`CID_MALL_WMS_DELAY_CONSUMER`

- Topic：`mallwms-delay-message`，订阅全部 tag。

- `DelayMessageListener` 遍历 Spring 注入的 `List<DelayHandler>` 调用 `match` + `handle`：

- `FpxMailChangeDelayHandler`

- `FpxOrderOutDelayHandler`

- `FpxAsyncCancelPostHandleDelayHandler`

- 失败返回 `RECONSUME_LATER`，由 RocketMQ 重投。

  

### 3. 平台消息中间件

- `WhcCommonSendRequest` / `WhcCommonSendResponse`（PAC SDK）：模块层面对外的「消息上抛」契约。

- `MessageSenderService`（HSF）：兜底/通用消息发送。

  

---

  

## 七、缓存（Redis / Tair）

  

### 1. 双 JedisPool 配置

`RdbConfiguration` 注册两个 Bean：`JedisPool`（旧实例）与 `NewJedisPool`（新实例），从 `operate.redis.*` / `operate.new.redis.*` 配置取值，连接池 `maxTotal=1000`、`maxWaitMillis=1000`、`timeout=3000ms`。

  

### 2. `ConcurrentLockSupport`

统一封装 `get` / `setex` / `setnx` / `ttl` / `del` / 分布式锁等：

- 通过 SwitchCenter `WhcCommonSendConfig.enableNewRedisRead` 控制读新/旧实例。

- `enableNewRedisReadBackOld=true` 时，新实例失败或返回空回退到旧实例。

- 分布式锁基于 `SET key value NX EX seconds`，返回值为 `OK` 视为成功（常量 `LOCK_SUCCESS = "OK"`）。

  

### 3. 业务缓存使用点

  

**`OwnerConvertManager`**（货主映射缓存）：

- `CACHE_KEY_MAPPING_BY_WMS_OWNER + wmsOwnerId`：缓存 wmsOwnerId → 4PX customerId/Code 映射。

- `CACHE_KEY_WMS_OWNER_ID_BY_OWNER_KEY + ownerKey`：缓存 ownerKey → wmsOwnerId。

- 通过 `jedis.get` / `jedis.setex` 读写，TTL 由配置项控制。

  

**`FpxClearanceVerificationManager`**（清关身份 ID 唯一性）：

- 借助 `ConcurrentLockSupport` 校验/记录身份证号 hash，过期时间 `WhcCommonSendConfig.fpxValidateIdHashCodeExpireTimes`（默认 30 天）。

- 仓库白名单 `fpxValidateIdHashCodeWhList = [HKG103, HKG333]`、解锁状态 `fpxValidateIdHashCodeUnLockStatus = [-1]`。

  

**`ToolServiceImpl`**：使用 `ConcurrentLockSupport` 做运维工具的并发互斥。

  

> 模块内未直接使用 `@Cacheable` 等 Spring Cache 注解，所有缓存均显式通过 `ConcurrentLockSupport` 或 `JedisPool` 操作。

  

---

  

## 八、配置开关与扩展机制

  

- `WhcMallWmsConfig`：whc 消息订阅白名单、打包消息开关、出库 2C/2B 模式、出库草稿单模式、出库延迟处理、TKNTYPE 透传、地址/手机号兜底仓白名单等。

- `WhcCommonSendConfig`：4PX 测试 customerId 映射、`actionCode` 黑名单、tpCode 清关黑名单、身份 ID 校验仓白名单与 TTL、`fpxTpCodeMap`、错误单号、货品状态黑名单、新 Redis 灰度开关等。

- `OwnerConfig` / `WarehouseConfig` / `MethodNameConfig` / `AddressConfig`：method 名映射、仓库参数、地址兜底等。

- `IBizSolutionHandler` + `ExtensionRouter`：按 `bizSource` + `match()` 在多套实现间路由（FPX vs Default），便于多客户/多场景扩展。

- `LinkCallbackProvider` 注解：标识 PAC 接收服务回调入口。

  

---

  

## 九、关键调用链路（速查）

  

1. **4PX → mallwms 入库单创建**

`WHC_COMMON_RECEIVE` → `WhcReceiveServiceImpl.invoke` → `convertMethodName=entry.order.create` → `EntryOrderCreate.invoke` → `WhcInterfaceWrapper.WhcCreateService` → 写入 cainiao_whc。

  

2. **mallwms → 4PX 入库收货回传**

WHC MetaQ tag=`order-receive-inbound` → `WhcMetaqListener` → `EntryOrderReceiveStrategy` → `IEntryOrderReceiveBizHandler`（默认/FPX 扩展）→ `SendSolution.sendMessage` → `ConciseLinkClient` `WHC_COMMON_SEND`（`toCode=fpx-gateway-wms-common`）。

  

3. **货品定时同步**

SchedulerX `GoodsSyncProcessor` → `GoodsManager.batchSyncGoodsFrom4Px` → `GoodsPullStrategy`（`API_PULL_GOODS_DATA`）→ Link 下发 4PX → 回写 cnitem。

  

4. **运单变更补偿**

接收端 `UpdateConsignOrderMailInfo` 处理失败/有约束 → `DelayMessageProducer.sendFpxChangeMailMq`（`mallwms-delay-message`，tag `FPX_MAIL_CHANGE`，10s）→ `DelayMessageListener` → `FpxMailChangeDelayHandler` 重试。

  

5. **出库清关校验**

`CustomsClearanceServiceImpl.verification` → `CustomsClearanceVerificationStrategy` → `FpxClearanceVerificationManager`（Redis 校验身份ID）→ Link 下发 4PX。

  

---

  

> 以上信息均来自工程当前源码（`whc-gateway-{client,common,infra,dal,biz,service}` 各模块），如需进一步细化到某一条策略 / handler 的字段映射，可基于对应类继续深入。