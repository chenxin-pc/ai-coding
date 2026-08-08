第一部分：
1、供应文档链接：
1. UU跑腿开放平台接入流程：[https://open.uupt.com/#/development/guide?t=%E6%8E%A5%E5%85%A5%E6%B5%81%E7%A8%8B&index=1](https://open.uupt.com/#/development/guide?t=%E6%8E%A5%E5%85%A5%E6%B5%81%E7%A8%8B&index=1)
2. UU跑腿开放平台接口文档：[https://open.uupt.com/#/development/apiSpecification?t=API%E8%B0%83%E7%94%A8%E8%A7%84%E8%8C%83&index=3-1-1](https://open.uupt.com/#/development/apiSpecification?t=API%E8%B0%83%E7%94%A8%E8%A7%84%E8%8C%83&index=3-1-1)
3. UU跑腿开放平台联调说明：[https://open.uupt.com/#/development/TestWay?t=%E8%81%94%E8%B0%83%E8%AF%B4%E6%98%8E&index=4](https://open.uupt.com/#/development/TestWay?t=%E8%81%94%E8%B0%83%E8%AF%B4%E6%98%8E&index=4)
4. 验签**文档地址**：[https://open.uupt.com/#/development/signAlgorithm?t=%E7%AD%BE%E5%90%8D%E7%AE%97%E6%B3%95&index=3-1-2](https://open.uupt.com/#/development/signAlgorithm?t=%E7%AD%BE%E5%90%8D%E7%AE%97%E6%B3%95&index=3-1-2)

2、需求背景：
外请用人是 TMS 运输管理系统中用于外请临时用工的核心模块，目前存在以下痛点：

- **平台单一**：线上仅接入搬运帮一家用人平台，议价能力和服务保障受限；
- **线下风险**：部分业务场景依赖线下人工对接，缺乏系统化管控，存在合规与安全风险；
- **搬运帮迭代缓慢**：搬运帮系统优化推进进度不理想，无法及时响应一线业务变化。

为降低单一平台依赖风险、保障一线外请用人业务连续性与稳定性，亟需接入第二家线上用人平台——**UU跑腿**，形成双平台互为备份的运力保障体系。

3、需求期望：
接入UU跑腿系统，完成订单的线上流转及费用结算。

4、业务方：
外运产品处-侯天宇

5、需要对接的接口：
主调接口： 
|   |
|---|
|[https://open.uupt.com/#/development/apiDoc/orderInterfaceList/orderPrice?businessGroup=1&t=%E5%9C%B0%E5%9D%80%E8%AF%A2%E4%BB%B7&index=3-2-3](https://open.uupt.com/#/development/apiDoc/orderInterfaceList/orderPrice?businessGroup=1&t=%E5%9C%B0%E5%9D%80%E8%AF%A2%E4%BB%B7&index=3-2-3)|
|[https://open.uupt.com/#/development/apiDoc/orderInterfaceList/addOrder?businessGroup=1&t=%E5%9C%B0%E5%9D%80%E5%8F%91%E5%8D%95&index=3-2-4](https://open.uupt.com/#/development/apiDoc/orderInterfaceList/addOrder?businessGroup=1&t=%E5%9C%B0%E5%9D%80%E5%8F%91%E5%8D%95&index=3-2-4)|
|[https://open.uupt.com/#/development/apiDoc/orderInterfaceList/addGratuity?businessGroup=1&t=%E5%8A%A0%E5%B0%8F%E8%B4%B9&index=3-2-5](https://open.uupt.com/#/development/apiDoc/orderInterfaceList/addGratuity?businessGroup=1&t=%E5%8A%A0%E5%B0%8F%E8%B4%B9&index=3-2-5)|
|[https://open.uupt.com/#/development/apiDoc/orderInterfaceList/orderDetail?businessGroup=1&t=%E6%9F%A5%E8%AF%A2%E8%AE%A2%E5%8D%95%E8%AF%A6%E6%83%85&index=3-2-6](https://open.uupt.com/#/development/apiDoc/orderInterfaceList/orderDetail?businessGroup=1&t=%E6%9F%A5%E8%AF%A2%E8%AE%A2%E5%8D%95%E8%AF%A6%E6%83%85&index=3-2-6)|
|[https://open.uupt.com/#/development/apiDoc/orderInterfaceList/cancelFee?businessGroup=1&t=%E6%9F%A5%E8%AF%A2%E5%8F%96%E6%B6%88%E8%AE%A2%E5%8D%95%E8%B4%B9%E7%94%A8&index=3-2-8](https://open.uupt.com/#/development/apiDoc/orderInterfaceList/cancelFee?businessGroup=1&t=%E6%9F%A5%E8%AF%A2%E5%8F%96%E6%B6%88%E8%AE%A2%E5%8D%95%E8%B4%B9%E7%94%A8&index=3-2-8)|
|[https://open.uupt.com/#/development/apiDoc/orderInterfaceList/cancelOrder?businessGroup=1&t=%E5%8F%96%E6%B6%88%E8%AE%A2%E5%8D%95&index=3-2-9](https://open.uupt.com/#/development/apiDoc/orderInterfaceList/cancelOrder?businessGroup=1&t=%E5%8F%96%E6%B6%88%E8%AE%A2%E5%8D%95&index=3-2-9)|
|[https://open.uupt.com/#/development/apiDoc/rechargeInterfaceList/account?businessGroup=3&t=%E6%9F%A5%E8%AF%A2%E8%B4%A6%E6%88%B7%E4%BD%99%E9%A2%9D&index=3-4-3](https://open.uupt.com/#/development/apiDoc/rechargeInterfaceList/account?businessGroup=3&t=%E6%9F%A5%E8%AF%A2%E8%B4%A6%E6%88%B7%E4%BD%99%E9%A2%9D&index=3-4-3)|
回调接口： 
[https://open.uupt.com/#/development/apiDoc/orderInterfaceList/callback?businessGroup=1&t=%E8%AE%A2%E5%8D%95%E7%8A%B6%E6%80%81%E5%9B%9E%E8%B0%83&index=3-2-10](https://open.uupt.com/#/development/apiDoc/orderInterfaceList/callback?businessGroup=1&t=%E8%AE%A2%E5%8D%95%E7%8A%B6%E6%80%81%E5%9B%9E%E8%B0%83&index=3-2-10)

第二部分:
1、主调接口确认安全规则：
主调不需要白名单
主调需要验签，从供应商文档中获取
主调不需要加解密
主调不需要token

2、回调接口确认安全规则 
主调保持一致

第三部分：
1、确认安全规则的来源（主调和回调需要分别说明）
例如：供应商文档

后续确认对应内容即可

注意： 
1、文档内容最好越具体越好，多个文档需要指出验签文档、接口文档
2、不确认供应商文档是否有加解密、验签、token规则，可以让AI先行分析作为参考




