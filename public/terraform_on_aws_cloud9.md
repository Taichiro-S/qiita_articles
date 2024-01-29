---
title: AWS Cloud9にTerraformの実行環境をつくる
tags:
  - AWS
  - cloud9
  - Terraform
private: false
updated_at: '2024-01-29T21:35:11+09:00'
id: e17651bddf321bf3a159
organization_url_name: null
slide: false
ignorePublish: false
---
## この記事を書いた経緯

ローカル環境からTerraformコマンドでAWSリソースを作成して遊んでいたら、急に反応がなくなってしまった...
Pluginがどうこうというエラーが出たり何も出なかったりで行き詰まってしまったので、**Cloud9にTerraform実行環境を作ってみる**ことに。
同じような症状で解決されている方がいたら、ぜひコメントで教えてください！

### 必要なもの

AWS アカウント
IAMユーザ（Terraform実行のためにはAdministratorAccessをポリシーに追加しておきます）

### 私の環境

 OS：macOS Ventura 13.2.1


**AWSのリソース作成により料金が発生する場合があります。自己責任でお願いします。**

## Cloud9にTerraform実行環境を構築

まずCloud9環境を作成します。
AWSのマネジメントコーソールからcloud9と検索 → 環境を作成。
<details open>
<summary>Cloud9の設定</summary>

- name：terraform
- Environment typeInfo：New EC2 instance
- Instance typeInfo：t2.micro (1 GiB RAM + 1 vCPU)
- platform：Amazon Linux 2
- Connection：AWS Systems Manager (SSM)
- VPC：デフォルト
- Subnet：パブリックサブネット

**デフォルトVPCがない場合は、VPC → アクション → デフォルトVPCを作成できます。**
</details>

Cloud9にデフォルトで入っているAWS CLIのバージョンがv1だったので、[公式のガイド](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html)に従ってv2にアップデートしておきます。

AWS CLI v2のパッケージをダウンロード → 解凍 → インストール → パスを通す。

``` shell
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install
$ echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
```

`source ~/.bashrc`で変更適用を忘れずに！

----
次に、Terraformのインストールです。
まずはtfenv（Terraformのバージョン管理ツール）をインストールしてパスを通します。

``` shell
$ git clone https://github.com/tfutils/tfenv.git ~/.tfenv
$ sudo ln -s ~/.tfenv/bin/* /usr/local/bin
```

`tfenv -v`でバージョンが表示されればOKです。

`tfenv list-remote`でインストール可能なバージョンの一覧を確認し、最新のものをインストールしましょう。
今回は1.4.6をインストールします。

``` shell
$ tfenv install 1.4.6
$ tfenv use 1.4.6
$ terraform version
Terraform v1.4.6
on linux_amd64
```

インストールされていることが確認できました。

## EC２インスタンスの起動

### AWSクレデンシャルの読み込み

まずはIAMユーザのクレデンシャルをterraformから参照できるようにします。
※profileを指定せずに実行すると、**Cloud9に設定されているデフォルトユーザ（AMTC）で実行される**みたいです（[こちら](https://dev.classmethod.jp/articles/aws-cloud9-aws-managed-temporary-credentials/)を参照）

[こちら](https://qiita.com/Hikosaburou/items/1d3765d85d5398e3763f)で紹介されているようにいくつか方法があるみたいです。
今回は一番手軽そうな`.tfvars`にprofileを書き込むやり方でいきます。

Terraformの作業ディレクトリとして`terraform`ディレクトリを作成して、`terraform.tfvars`を作成します。
AWSマネジメントコンソールで、IAM → 作成したユーザ → セキュリティ認証情報 → アクセスキーを作成でアクセスキーを作成します
**アクセスキーの値は一度表示を消すと二度と見れなくなるので、csvファイルをダウンロードするのを忘れずに！**

以下のように`terraform.tfvars`に認証情報を書き込みます。

```tfvars:terraform.tfvars
access_key = "***********"
secret_key = "*******************"
```


<span style="color: red; ">このファイルはgithubにアップロードしないように`.gitignore`ファイルに記載しておきましょう。</span>

`.gitignore`ファイルは[こちら](https://www.toptal.com/developers/gitignore)からterraformと入力して簡単に作成することができます。

<details open>
<summary>こんな感じになります。</summary>

```text:.gitignore
# Created by https://www.toptal.com/developers/gitignore/api/terraform
# Edit at https://www.toptal.com/developers/gitignore?templates=terraform

### Terraform ###
# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log
crash.*.log

# Exclude all .tfvars files, which are likely to contain sensitive data, such as
# password, private keys, and other secrets. These should not be part of version
# control as they are data points which are potentially sensitive and subject
# to change depending on the environment.
*.tfvars
*.tfvars.json

# Ignore override files as they are usually used to override resources locally and so
# are not checked in
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Include override files you do wish to add to version control using negated pattern
# !example_override.tf

# Include tfplan files to ignore the plan output of command: terraform plan -out=tfplan
# example: *tfplan*

# Ignore CLI configuration files
.terraformrc
terraform.rc

# End of https://www.toptal.com/developers/gitignore/api/terraform
```

</details>

### EC2インスタンス作成

次に`main.tf`を作成し、コードを実行するprofileの設定、変数の定義を記述します。

```tf:main.tf
provider "aws" {
  region     = "ap-northeast-1"
  access_key = var.access_key
  secret_key = var.secret_key
}

variable "access_key" {
  type = string
}

variable "secret_key" {
  type = string
}
```

`terraform init`を実行して、`Terraform has been successfully initialized!`と表示されればOKです。

やっとEC2インスタンスの作成です。
`main.tf`に以下のように追記します。

```tf: main.tf
resource "aws_instance" "TF_C9" {
  ami           = "ami-0ce107ae7af2e92b5"
  instance_type = "t2.micro"
  tags = {
    Name = "TF_C9"
  }
}
```

`terraform plan` → `terraform apply -auto-approve`で実行します。

`Apply complete! Resources: 1 added, 0 changed, 0 destroyed.`と表示されればOKです。
マネジメントコンソールからもEC2インスタンスが確認できるはずです。

これで、Cloud9からTerraformを実行することができました！！


**作成したEC2インスタンスは`terraform destroy`しておきましょう**

## おまけ

ここからはオマケですが、ソースコードをCloud9からGithubにアップロードします。
Cloud9でSSHキーを生成 → githubに登録 → リポジトリにpush という流れです。

SSHキーを生成します
`cd ~/.ssh`  
`ssh-keygen`  

いくつか質問が出てきますが、とりあえず全てエンターで鍵を生成します。
`/.ssh/id_rsa.pub`にある公開鍵の中身をGithubのsettings → SSH keysで登録します。

リポジトリを作成し、Cloud9からリポジトリのSSHのURLを登録してpushします。

## おまけのおまけ

さらにおまけになりますが最後に、AWSクレデンシャルがGithubの公開リポジトリにリークする事故を防ぐためのツール、**git secrets**を導入します。

Cloud9にgit secretsをインストールします。
`git clone https://github.com/awslabs/git-secrets.git`
`cd git-secrets`
`sudo make install`

`terraform`ディレクトリでawsのアクセスキーのパターンを認識するオプションを設定します(`--global`オプションでglobalに設定しても良いです)
`git secrets --register-aws`

プロジェクトディレクトリで`git init`を実行し、`git secrets --install`で対象のリポジトリに対して有効化しようとすると、以下のようなエラーが...

``` shell
/usr/local/bin/git-secrets: line 208: say: command not found
/usr/local/bin/git-secrets: line 208: say: command not found
/usr/local/bin/git-secrets: line 208: say: command not found
```

`say`はmacOSで使われる読み上げのコマンドらしく、今回使用しているAmazon Linux 2では使えないみたいです。
該当ファイルを開き、208行目の say を echo に書きかえます。
`sudo vi /usr/local/bin/git-secrets`

```text:git-secrets
　...
  chmod +x "${dest}"
  echo "$(tput setaf 2)✓$(tput sgr 0) Installed ${hook} hook to ${dest}" # ここを変更
}
```

なんでここだけ`say`になってるのかは謎です...

適当なテキストファイルにawsのアクセスキーを書き込み、commitしてみます。
以下のようにエラーが表示され、コミットが拒否されるはずです。

``` shell
test.txt:1:access_key = "***********"
test.txt:2:secret_key = "******************"

[ERROR] Matched one or more prohibited patterns
```

`git log`でコミットされていないことが確認できます。

これでおっちょこちょいなワタシも安心！！

さらに詳しく知りたい方はこちらが参考になります。
https://zenn.dev/kkk777/articles/8f55db1e9678f2

### 最後に

初めて記事を書いてみました。4時間くらいかかりました。~~もう書きません。~~
自分用メモとは違い、全体の流れ、それぞれの操作をなんのためにやっているのか、ちゃんと言葉にして書くことで勉強になりました。

### 参考にさせていただいた記事

https://dev.classmethod.jp/articles/cloud9-terraform/
https://dev.classmethod.jp/articles/aws-cloud9-aws-managed-temporary-credentials/
https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html
https://kosuke-space.com/cloud9-github
https://qiita.com/Hikosaburou/items/1d3765d85d5398e3763f

