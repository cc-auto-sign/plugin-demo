# 插件demo

## 插件包结构设计

采用 **ZIP 格式** 打包，目录结构如下：

```bash
my_plugin/
├── manifest.yaml   # 必须 - 元数据清单
├── main.py         # 可选 - 插件入口主程序 (python插件)
├── main.js         # 可选 - 插件入口主程序 (nodejs插件)
├── requirements.txt       # 可选 - Python依赖声明
├── package.json           # 可选 - nodejs依赖声明
├── README.md              # 可选 - 文档
├── assets/                # 可选 - 静态资源
│    └── icon.png           # 可选 - 如不填则用name作为icon
└── signature.sig        # 必须 - 完整性校验文件

```

## 文件说明

### 1.**元数据清单 (manifest.yaml)**

```yaml
# 基础信息
plugin:
  id: "weibo_signin"                 # 全局唯一标识符 不可以和其他插件重复
  name: "微博自动签到插件"              # 展示名称
  version: "1.2.0"                   # 语义化版本号
  author: "dev@example.com"          # 开发者邮箱/ID
  description: "支持微博超话签到..."    # 简短描述
  type: 1|2|3                        # 1是curl，2是nodejs，3是python，目前只支持1  

# 用户配置时需填写的信息(在type为2或者3时 下面参数会作为环境变量以env_xxxx的形式方便开发者调用)
# 在type为1时可以用${env_xxxx}的形式调用
config:
  token:
  	type: "string"
  	label: '访问令牌'
  	required: true
  followMode:
  	type: "select"
  	label: "关注模式"
  	required: true
  	options: 
      - value: "all"
        label: "全部"
      - value: "new"
        label: "最新"
  
curl: # curl运行命令，在type为1时生效
  command: "curl https://www.baidu.com"
  
# 依赖声明
requirements:
  min_system_version: "1.0.0"        # 最低系统版本要求
  
# nodejs运行和python运行目前想法是沙箱形式运行，参考dify的沙箱
nodejs: # 在type为2时生效 （预留，暂不可用）
  version: ">26" 
  
python: # 在type为3时生效（预留，暂不可用）
  version: ">3.8"

# 安全信息 （暂不可以，预留）
security:
  allow_apis: ["http_request"]       # 声明所需的沙箱权限，如网络访问等。（暂不可以）
  
# 钩子机制 (目前不支持，后续支持，nodejs是2就用node运行，是3就用python运行)
hooks:
  before_install: "pre_install.py"  # 安装前运行
  after_uninstall: "cleanup.py"     # 卸载后清理
  
```

### 2. 插件打包签名

#### 2.1 生成密钥对

```bash
# 生成密钥对（开发者执行）
openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:4096
openssl rsa -pubout -in private_key.pem -out public_key.pem
# 注: 需要把生成的公钥上传至插件发布服务器
```

#### 2.2 生成签名

```bash
# 生成签名（开发者执行）
openssl dgst -sha256 -sign private_key.pem -out signature.sig plugin_package.zip
```

#### 2.3 签名文件追加到压缩包内 (发布时上传该压缩包即可)

```bash
zip -u plugin_package.zip signature.sig
```

#### 2.4 插件服务器服务端验证流程

在插件上传时会用用户上传到服务器上的公钥进行校验

````bash
# 服务端校验步骤
# 1. 从zip包中抽取签名文件
unzip -p plugin_package.zip signature.sig > extracted_signature.sig
# 2. 拷贝未签名的原始内容（移除临时签名）
zip -d plugin_package.zip signature.sig
# 3. 对去除签名的文件进行校验
openssl dgst -sha256 \
  -verify public_key.pem \
  -signature extracted_signature.sig \
  original_package.zip
# 期望输出结果："Verified OK"
````

校验通过后会进行插件审核，审核通过的插件用户可以在插件商店中查看安装。

