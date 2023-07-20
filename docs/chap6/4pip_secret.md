# **4 如何将Secrets扫描加入到GitLab Pipeline**

![Alt Image Text](../images/chap6_4_n1.png "Body image")

**概述**

如果要管理应用程序开发管道以交付软件，则需要实施一项关键安全控制。您需要确保您的机密受到保护。简单地说，机密是您需要保护的凭据，因为该凭据具有特权功能。例如：


* 用户名和密码组合
* 接口密钥
* JSON 网络令牌
* 私钥

开发人员经常意外地将这些凭据保存到存储库中。将这些凭据或机密添加到存储库后，清理可能会很痛苦，并且需要您更改凭据。更好的长期方法是使用“提交前检查”来避免存储机密。该主题将在另一篇文章中介绍。本博客重点介绍如何扫描现有存储库以查找 Gitlab 管道中存储的机密。

### **识别或创建存储库**

如果您想继续操作，可以单击此链接获取公共仓库。

[`https://gitlab.com/cmg_public/secrets_scanning_test`](https://gitlab.com/cmg_public/secrets_scanning_test)

第一步是确定目标存储库。如果您正在入门但没有，只需在 Gitlab 中创建一个新项目即可。

**从菜单中单击“项目>创建新项目>导入上面项目地址”。**

![Alt Image Text](../images/chap6_4_n2.png "Body image")

扫描规则配置文件

文件路径：`.gitlab/secret-detection-ruleset.toml`

```
[secrets]
description = 'CMG DevSecOps Crash Course, secrets custom rules configuration'

[[secrets.passthrough]]
type = "raw"
target = "gitleaks.toml"
value = """\
title = "gitleaks config"
# Add regexes to the regex table
[[rules]]
description = "Test for Raw Custom Rulesets"
regex = '''Custom Raw Ruleset T[est]{3}'''

[[rules]]
description = "CMG Custom Generic Password"
regex = '''(?i)(password|passw)'''
tags = ["key", "Custom Password", "generic"]
"""
```

您会注意到我添加了一些自定义规则。这些将用于标识默认情况下未启用的某些类型的机密。

您可以根据需要创建任意数量的规则，以查找通常出现在开发人员代码中的模式。

"CMG Generic JWT" 此正则表达式 （REGEX） 查找存储在文件中的 JSON Web Token。诸如`“JWT”`、`“jwt_token”`、`“secret”`、`“bearer”`等。正则表达式不区分大小写。

```
[[rules]]
description = "CMG Generic JWT"
regex = '''(?i)(jwt|jwt_token|secret|bearer)'''
tags = ["key", "JWT", "generic"]
```

"CMG Custom Generic Password" 此正则表达式查找存储在文件中的单词“password”的常见派生词。此正则表达式也不区分大小写。

```
[[rules]]
description = "CMG Custom Generic Password"
regex = '''(?i)(password|passw)'''
tags = ["key", "Custom Password", "generic"]
"""
```

**Pipeline配置文件**

此模板将用于根据默认设置和任何自定义正则表达式字符串（如我们在前面的步骤中添加的字符串）扫描机密。

```
# This file is a template, and might need editing before it works on your project.
# This template moved to Jobs/Secret-Detection.gitlab-ci.yml in GitLab 14.0
# Issue: https://gitlab.com/gitlab-org/gitlab/-/issues/292977
include:
  template: Jobs/Secret-Detection.gitlab-ci.yml
```

此时，我们已准备好运行管道;但是，我们在存储库中没有任何要扫描的内容。在下一步中，我们将创建一些用于测试配置的机密。

## 用于测试的Secret文件

* **`Fake.json`：我们可以用它来测试`JWT`**。

```
{
  "username": "SecurityBooBoo",
  "password": "Oh-Y3aH_CmG_2022-123"
}
```

* **FakePrivKey.key：我们可以使用此文件来测试私钥**。

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

* **FakeAPIKey.txt：我们可以用它来查找和测试 API 密钥**。

```
Apikey=33219b16f39fc40d3a1be5e9e6f56abe
```

### 运行测试并观察结果

默认情况下，每次添加并提交文件时，都会触发构建。如果单击 Web 界面左侧窗格中的“CI/CD”>“管道”对象，则可以看到每个作业的结果。

单击每个屏幕上的“运行管道”按钮。系统将提示您输入其他可以忽略的选项。

![Alt Image Text](../images/chap6_4_n3.png "Body image")

几秒钟后，管道作业将开始。**一旦它变成蓝色，您可以单击“secret_detection”阶段并观看输出**。

![Alt Image Text](../images/chap6_4_n4.png "Body image")

如果一切按计划进行，您应该会看到类似的文本。

![Alt Image Text](../images/chap6_4_n5.png "Body image")


## **下载结果**

几分钟后，结果将可供下载关联作业。单击作业右侧的“下载按钮”，然后选择“`secret_detection.secret_detection`”文件。

它将下载为 JSON。然后，您可以在自己喜欢的文本编辑器或在线 JSON 格式化工具中打开它。

![Alt Image Text](../images/chap6_4_n6.png "Body image")

查看结果

```
{
  "version":"14.0.4",
  "vulnerabilities":[
    {
      "id":"a70bbb54bc6faeadc71027dfb50203a1803cc10d221af9156b5393a211ccddd2",
      "category":"secret_detection",
      "name":"CMG Custom Generic Password",
      "message":"CMG Custom Generic Password detected; please remove and revoke it if this is a leak.",
      "description":"CMG Custom Generic Password",
      "cve":".gitlab/secret-detection-ruleset.toml:5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8:CMG Custom Generic Password",
      "severity":"Critical",
      "confidence":"Unknown",
      "raw_source_code_extract":"password",
      "scanner":{
        "id":"gitleaks",
        "name":"Gitleaks"
      },
      "location":{
        "file":".gitlab/secret-detection-ruleset.toml",
        "commit":{
          "sha":"0000000"
        },
        "start_line":19
      },
      "identifiers":[
        {
          "type":"gitleaks_rule_id",
          "name":"Gitleaks rule ID CMG Custom Generic Password",
          "value":"CMG Custom Generic Password"
        }
      ]
    },
    {
      "id":"4d5f0e089322ca90db4e7b3baa58a3c3a68bf63af20ec9afe2e1120feea93a95",
      "category":"secret_detection",
      "name":"CMG Custom Generic Password",
      "message":"CMG Custom Generic Password detected; please remove and revoke it if this is a leak.",
      "description":"CMG Custom Generic Password",
      "cve":".gitlab/secret-detection-ruleset.toml:e45d90957eec7387726c6a1b174da7b566a24ff4cb060dcbcdfebb931a93ffe3:CMG Custom Generic Password",
      "severity":"Critical",
      "confidence":"Unknown",
      "raw_source_code_extract":"passw",
      "scanner":{
        "id":"gitleaks",
        "name":"Gitleaks"
      },
      "location":{
        "file":".gitlab/secret-detection-ruleset.toml",
        "commit":{
          "sha":"0000000"
        },
        "start_line":19
      },
      "identifiers":[
        {
          "type":"gitleaks_rule_id",
          "name":"Gitleaks rule ID CMG Custom Generic Password",
          "value":"CMG Custom Generic Password"
        }
      ]
    },
    {
      "id":"95d9614ca06da0f4fc8f67319a44585ee6295a26060cd256655ae6f503e41ef2",
      "category":"secret_detection",
      "name":"CMG Custom Generic Password",
      "message":"CMG Custom Generic Password detected; please remove and revoke it if this is a leak.",
      "description":"CMG Custom Generic Password",
      "cve":".gitlab/secret-detection-ruleset.toml:e7cf3ef4f17c3999a94f2c6f612e8a888e5b1026878e4e19398b23bd38ec221a:CMG Custom Generic Password",
      "severity":"Critical",
      "confidence":"Unknown",
      "raw_source_code_extract":"Password",
      "scanner":{
      "id":"gitleaks",
      "name":"Gitleaks"
    },
      "location":{
      "file":".gitlab/secret-detection-ruleset.toml",
      "commit":{
      "sha":"0000000"
    },
      "start_line":18
    },
      "identifiers":[
      {
      "type":"gitleaks_rule_id",
      "name":"Gitleaks rule ID CMG Custom Generic Password",
      "value":"CMG Custom Generic Password"
    }
      ]
    },
      {
      "id":"52256d01ca112c54e5ca22e982b096852bf279f206f425aebf4421cfc8ac426e",
      "category":"secret_detection",
      "name":"CMG Custom Generic Password",
      "message":"CMG Custom Generic Password detected; please remove and revoke it if this is a leak.",
      "description":"CMG Custom Generic Password",
      "cve":".gitlab/secret-detection-ruleset.toml:e7cf3ef4f17c3999a94f2c6f612e8a888e5b1026878e4e19398b23bd38ec221a:CMG Custom Generic Password",
      "severity":"Critical",
      "confidence":"Unknown",
      "raw_source_code_extract":"Password",
      "scanner":{
      "id":"gitleaks",
      "name":"Gitleaks"
    },
      "location":{
      "file":".gitlab/secret-detection-ruleset.toml",
      "commit":{
      "sha":"0000000"
    },
      "start_line":20
    },
      "identifiers":[
      {
      "type":"gitleaks_rule_id",
      "name":"Gitleaks rule ID CMG Custom Generic Password",
      "value":"CMG Custom Generic Password"
    }
      ]
    },
      {
      "id":"f9c6b0ef5af1fb0298ff99fc6467e76fc44a4e6d1dc3699cfd8c7249eb293ad5",
      "category":"secret_detection",
      "name":"CMG Custom Generic Password",
      "message":"CMG Custom Generic Password detected; please remove and revoke it if this is a leak.",
      "description":"CMG Custom Generic Password",
      "cve":"Fake.json:5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8:CMG Custom Generic Password",
      "severity":"Critical",
      "confidence":"Unknown",
      "raw_source_code_extract":"password",
      "scanner":{
      "id":"gitleaks",
      "name":"Gitleaks"
    },
      "location":{
      "file":"Fake.json",
      "commit":{
      "sha":"0000000"
    },
      "start_line":3
    },
      "identifiers":[
      {
      "type":"gitleaks_rule_id",
      "name":"Gitleaks rule ID CMG Custom Generic Password",
      "value":"CMG Custom Generic Password"
    }
      ]
    }
      ],
      "scan":{
      "analyzer":{
      "id":"secrets",
      "name":"secrets",
      "url":"https://gitlab.com/gitlab-org/security-products/analyzers/secrets",
      "vendor":{
      "name":"GitLab"
    },
      "version":"4.5.4"
    },
      "scanner":{
      "id":"gitleaks",
      "name":"Gitleaks",
      "url":"https://github.com/zricethezav/gitleaks",
      "vendor":{
      "name":"GitLab"
    },
      "version":"8.15.0"
    },
      "type":"secret_detection",
      "start_time":"2022-12-30T02:08:44",
      "end_time":"2022-12-30T02:08:44",
      "status":"success"
    }
    }
```
 
![Alt Image Text](../images/chap6_4_n7.png "Body image")
 
 查看漏洞报告
 
 ![Alt Image Text](../images/chap6_4_n8.png "Body image")

这个例子找到了我们的“Fake.Json”文件。在详细信息页面上，单击"Fake.json:3". 该工具会将您定向到找到的特定文件和凭据：

 ![Alt Image Text](../images/chap6_4_n9.png "Body image")


## 总结

总之，从安全角度来看，您可以看到这是简单且有价值的。一旦你的秘密被提交，修复起来就是一个痛苦的过程。这是一种方法，但不会否定对凭据和机密存储进行“预提交扫描”或使用“vault”的需要。