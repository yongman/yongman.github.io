---
title: OneAPI 充值和使用流程
date: 2023-12-31 23:58:30
---

1. 首页点击充值，需要先注册并登录

![](https://raw.githubusercontent.com/yongman/i/picgo/202401010036126.png)

选择要充值的金额。更多充值金额可以点击。[https://v6.xiking.win:3343](https://v6.xiking.win:3343)

2. 填写邮箱获取兑换卡密

![](https://raw.githubusercontent.com/yongman/i/picgo/202401010047211.png)

订单完成后会自动展示卡密内容。

![](https://raw.githubusercontent.com/yongman/i/picgo/202401010048920.png)

3. 兑换卡密

点击 [https://v6.xiking.win:3443/topup](https://v6.xiking.win:3443/topup) 充值，输入兑换码点击兑换，充值到账号。

![](https://raw.githubusercontent.com/yongman/i/picgo/202401010100709.png)

4. 新建令牌 APIKEY

点击 [https://v6.xiking.win:3443/token](https://v6.xiking.win:3443/token) 新建令牌，也就是 APIKEY，可以创建多个令牌区分不同的应用。创建成功后点击对应令牌后面的复制即可复制令牌到剪贴板。

![](https://raw.githubusercontent.com/yongman/i/picgo/202401010102529.png)

5. 接入应用，修改接入域名和 APIKEY

- 接口地址：

请把官方的API接口域名 `api.openai.com` 更换为 `v6.xiking.win:3443` 即 `https://v6.xiking.win:3443/v1/chat/completions`

请求参数同 OpenAI 官网 API 文档一致。

- APIKEY获取：

APIKEY 就是步骤4中获取到的令牌。