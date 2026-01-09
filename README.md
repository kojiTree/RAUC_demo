# RAUC_demo

このリポジトリは、**RAUCの本質（Active artifact の検証と適用）を理解するための最小デモ**です。

## このデモで分かること

- RAUCは「同期ツール」ではない
- RAUCは「署名された確定物（Active）を適用するエンジン」である
- 通信・同期・編集はRAUCの管轄外である

## このデモで扱わないこと

- Draft設定の同期
- ロボット↔サーバ通信
- 設定ファイルの直接編集・展開

---

# デモ手順:server側

## 1. インストール

```bash
sudo apt update
sudo apt install -y rauc openssl squashfs-tools e2fsprogs
```

### この時点で存在するファイル

* 特になし（ツールがインストールされるだけ）

### これは何を意味するか

* RAUC を使うための **道具が揃った状態**
* 更新対象・設定・データはまだ一切登場していない

### 状態の言い換え

* **準備段階**
* 何かを「作った」わけではない

---

## 2. 署名鍵の作成

```bash
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout demo.key.pem \
  -out demo.cert.pem \
  -subj "/O=rauc Inc./CN=rauc-demo"
```

### 生まれるファイル

```text
demo.key.pem   （秘密鍵）
demo.cert.pem  （公開証明書）
```

### それぞれは何か

#### demo.key.pem

* ファイルに **署名を付けるための鍵**
* この鍵を持っている人だけが「正しい更新ファイル」を作れる
* 他人に渡すと「誰でも正規品を名乗れる」ので、厳重管理が必要

#### demo.cert.pem

* 「どの署名を信用するか」を判断するための証明書
* 更新を受け取る側（ロボット側）に置く
* これがあることで「これは信頼できる作成者のものか？」を確認できる

### これは何を意味するか

* **更新ファイルの“身分証明”の仕組みを作った**
* まだ更新内容は何も決まっていない

---

## 3. 「更新対象」を作る（rootfsイメージ）

```bash
mkdir -p rootfs_v1/etc
echo "v1" > rootfs_v1/etc/demo-version
```

```bash
dd if=/dev/zero of=rootfs.ext4.img bs=1M count=64
mkfs.ext4 -F rootfs.ext4.img

mkdir -p mnt_rootfs
sudo mount -o loop rootfs.ext4.img mnt_rootfs
sudo cp -a rootfs_v1/* mnt_rootfs/
sync
sudo umount mnt_rootfs
```

### 生まれるファイル

```text
rootfs.ext4.img
```

### これは何か

* **更新したい中身そのもの**
* 今回は rootfs だが、RAUCにとっては：

  * 中身の意味は分からない
  * ただのバイナリファイル

### 重要なポイント

* この時点では：

  * 配るかどうかは決まっていない
  * 何度作り直してもよい
  * 編集してもよい

### これは何を意味するか

* **「将来、配るかもしれない中身」**
* まだ確定していない
* ただの素材

---

## 4. RAUC bundle を作る

```bash
mkdir -p bundle_dir
cp rootfs.ext4.img bundle_dir/
```

```bash
cat > bundle_dir/manifest.raucm << 'EOF'
[update]
compatible=rauc-demo-x86
version=1.0

[bundle]
format=plain

[image.rootfs]
filename=rootfs.ext4.img
EOF
```

```bash
rauc --cert demo.cert.pem --key demo.key.pem bundle bundle_dir update-1.0.raucb
rauc info update-1.0.raucb
```

### 生まれるファイル

```text
manifest.raucm
update-1.0.raucb
```

---

### manifest.raucm は何か

* 「この更新ファイルはどういう構成か」を書いた **説明書**
* 内容としては：

  * どの環境向けか（compatible）
  * どのファイルが含まれているか
  * それらの名前

### 意味

* 人間とRAUCの間で共有される **構成の宣言**
* これを元にRAUCが検証・処理を行う

---

### update-1.0.raucb は何か（最重要）

#### 中身の正体

`update-1.0.raucb` は次をひとまとめにしたもの：

* rootfs.ext4.img
* manifest.raucm
* 各ファイルのチェックサム
* 作成者の署名

#### これは何を意味するか

* **中身が完全に確定した更新ファイル**
* 1バイトでも書き換えると：

  * チェックサムが合わない
  * 署名検証に失敗する

#### 性質

* もう編集しない
* コピーして配るだけ
* 正しいかどうかは機械的に検証できる

---

# デモ手順:client側

（write-slot 前提・最短版）

> この章では、server 側で作成された更新用イメージを
> **RAUC を使って指定された slot に書き込む**ところまでを行う。

---

## 前提

* server 側で以下が既に用意されている

  * `rootfs.ext4.img`（更新したい中身）
* client 側で以下がインストールされている

```bash
sudo apt update
sudo apt install -y rauc rauc-service
```

---

## 1. 書き込み先（slot）を用意する

### やっていること

* 実機パーティションの代わりに **loopback イメージを slot として用意**する

```bash
dd if=/dev/zero of=slot.img bs=1M count=64
mkfs.ext4 -F slot.img

sudo losetup -fP slot.img
losetup -a | grep slot.img
```

### 期待される出力（例）

```
/dev/loop0: (/home/koji/halna_server/RAUC_demo/slot.img)
```

---

## 2. RAUC の最小設定を行う

### やっていること

* 「どこに書き込むか」を RAUC に教える

```bash
sudo mkdir -p /etc/rauc
sudo tee /etc/rauc/system.conf > /dev/null << 'EOF'
[system]
compatible=demo
bootloader=noop

[slot.test.0]
type=ext4
device=/dev/loop0
EOF
```

---

## 3. RAUC service を起動する

### やっていること

* write-slot 実行のために RAUC のサービスを起動

```bash
sudo systemctl start rauc
```

---

## 4. 更新イメージを書き込む（write-slot）

### やっていること

* 指定された slot に、指定されたイメージを **そのまま書き込む**

```bash
sudo rauc write-slot test.0 bundle_dir/rootfs.ext4.img
```

※ `bundle_dir/rootfs.ext4.img` は実際の配置場所に合わせる

### 成功時の挙動

* エラーが出ない
* 数秒でプロンプトが戻る

---

## 5. 書き込み結果を確認する

### やっていること

* 本当に書き換わったかを人間が確認する

```bash
sudo mount /dev/loop0 /mnt
cat /mnt/etc/demo-version
sudo umount /mnt
```

### 期待される出力

```
v1
```

---

## この手順で何をやっていたか（要約）

* RAUC に

  * 「書き込み先（slot）」を設定し
  * 「書き込む中身」を渡し
* **決められた場所に、決められた中身を書かせた**

---

## この手順で確認できたこと

* RAUC は

  * system.conf を正しく読み
  * slot を認識し
  * 指定されたデバイスに
  * 指定されたイメージを書き込める
* 通信・取得・同期は RAUC の責務ではない
* install / 起動切替がなくても
  **「適用エンジン」としての動作は確認できる**



