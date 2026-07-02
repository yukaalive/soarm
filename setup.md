# SO-101 ロボットアーム組み立てメモ

<!-- TOC tocDepth:2..3 chapterDepth:2..6 -->

- [1. ソフトウェアの事前準備](#1-ソフトウェアの事前準備)
- [2. モーターの情報取得と設定](#2-モーターの情報取得と設定)
    - [2.1. 各モーターバスに関連付けられたUSBポートを見つける](#21-各モーターバスに関連付けられたusbポートを見つける)
    - [2.2. モーターの事前準備](#22-モーターの事前準備)
    - [2.3. モーターの初期化](#23-モーターの初期化)
- [3. ロボットアームの組み立て](#3-ロボットアームの組み立て)
- [4. キャリブレーション（校正）](#4-キャリブレーション校正)
    - [4.1. キャリブレーションデータ（計測データ）の実際の例](#41-キャリブレーションデータ計測データの実際の例)
- [5. テレオペレーションの実行（カメラなし）](#5-テレオペレーションの実行カメラなし)
- [6. カメラの検索](#6-カメラの検索)
- [7. テレオペレーションの実行（カメラあり）](#7-テレオペレーションの実行カメラあり)
- [8. 学習データの記録（Hugging Faceへのアップロードなし）](#8-学習データの記録hugging-faceへのアップロードなし)
- [9. 学習データの記録（Hugging Faceへのアップロードあり）](#9-学習データの記録hugging-faceへのアップロードあり)
- [10. 学習の実施](#10-学習の実施)
    - [10.1. 事前準備](#101-事前準備)
    - [10.2. SageMaker AIのノートブックインスタンスで学習する場合の環境構築](#102-sagemaker-aiのノートブックインスタンスで学習する場合の環境構築)
    - [10.3. EC2インスタンスで学習する場合の環境構築](#103-ec2インスタンスで学習する場合の環境構築)
    - [10.4. 学習実行](#104-学習実行)
- [11. 推論の実施](#11-推論の実施)

<!-- /TOC -->

## 1. ソフトウェアの事前準備

MinicondaやMiniforgeなどがインストールされたPCで、conda仮想環境を作成

```sh
conda create -y -n lerobot python=3.10
```

conda仮想環境を有効化（activate）

```sh
conda activate lerobot
```

ffmpegを仮想環境にインストール

```sh
conda install ffmpeg -c conda-forge
```

lerobotリポジトリをcloneし、v0.4.2に切り替える

```sh
git clone https://github.com/huggingface/lerobot.git
cd lerobot
git checkout v0.4.2
```

ライブラリを仮想環境にインストール

```sh
pip install -e .
pip install -e ".[feetech]"
```

## 2. モーターの情報取得と設定

### 2.1. 各モーターバスに関連付けられたUSBポートを見つける

以下の手順で、各アームのモーター制御ボード（モーターバス）をUSB経由でPCに接続し、それぞれのシリアルポート名を調べる  
※リーダーとフォロワー2つのボードがあるので、どちらのボードをどのアーム用とするかを自分で決める必要がある

1. 片方のボードだけUSB経由でPCに接続（電源もボードに接続）
2. 以下のコマンドを実行

    ```sh
    lerobot-find-port
    ```

    以下のような表示がされる

    ```stdout
    Finding all available ports for the MotorsBus.
    Ports before disconnecting: [...]
    Remove the USB cable from your MotorsBus and press Enter when done.
    ```

3. モーターバスのUSBケーブルの接続を外してEnterキーを押すと、以下のように表示されるのでメモしておく

    ```stdout
    The port of this MotorsBus is '/dev/tty.usbmodemXXXXXXXXXXX'
    ```

4. もう片方のボードも同様に実施

#### 2.1.1. 実際に取得したシリアルポート名

- Leader

    ```stdout
        The port of this MotorsBus is '/dev/tty.usbmodem5AB90678131'
    ```

- Follower

    ```stdout
        The port of this MotorsBus is '/dev/tty.usbmodem5AB90672311'
    ```

### 2.2. モーターの事前準備

リーダーアームとフォロワーアームに使用するモーターを特定する。各モーターにID1～ID6を識別できるシールなどを貼り付けておくとよい

- フォロワーアームに使う6つのモーターはすべて同じ減速比（1 / 345）のモデル
  - Pro kitの場合はFeetech STS3215-C018/C047, そうでない場合はFeetech STS3215-C001
- リーダーアームには異なるギア比を持つモーターを使用

| リーダーアーム軸      | モーターID | 減速比  | 型番                 |
| --------------------- | ---------- | ------- | -------------------- |
| ベース/ショルダーヨー | 1          | 1 / 191 | Feetech STS3215-C044 |
| ショルダーピッチ      | 2          | 1 / 345 | Feetech STS3215-C001 |
| エルボー              | 3          | 1 / 191 | Feetech STS3215-C044 |
| リストロール          | 4          | 1 / 147 | Feetech STS3215-C046 |
| リストピッチ          | 5          | 1 / 147 | Feetech STS3215-C046 |
| グリッパー            | 6          | 1 / 147 | Feetech STS3215-C046 |

### 2.3. モーターの初期化

以下の手順で、各モーターを初期化（モーターのメモリにモーターIDと通信速度を設定）する  
※リーダーアーム用のモーターをID6からID1の順に設定した後、フォロワーアーム用のモーターも同様に設定

1. まずは、リーダーアーム用のモーター制御ボード（モーターバス）をUSB経由でPCに接続
2. 以下のコマンドを実行（teleop.portは上記で取得したシリアルポート名を使用）

    ```sh
    lerobot-setup-motors \
        --teleop.type=so101_leader \
        --teleop.port=/dev/tty.usbmodem5AB90678131
    ```

    以下のような表示がされる

    ```stdout
    Connect the controller board to the 'gripper' motor only and press enter.
    ```

3. リーダーアーム用のID6のモーターを3ピンケーブルでモーターバスに接続してEnterキーを押す

    以下のような表示がされる

    ```stdout
    'gripper' motor id set to 6
    Connect the controller board to the 'wrist_roll' motor only and press enter.
    ```

4. 続けてID5～ID1の順に同様にモーターを1つずつモーターバスに接続してEnterキーを押す

    以下のような表示がされる

    ```stdout
    'wrist_roll' motor id set to 5
    Connect the controller board to the 'wrist_flex' motor only and press enter.
    'wrist_flex' motor id set to 4
    Connect the controller board to the 'elbow_flex' motor only and press enter.
    'elbow_flex' motor id set to 3
    Connect the controller board to the 'shoulder_lift' motor only and press enter.
    'shoulder_lift' motor id set to 2
    Connect the controller board to the 'shoulder_pan' motor only and press enter.
    'shoulder_pan' motor id set to 1
    ```

5. 続いて、フォロワーアーム用のモーター制御ボード（モーターバス）をUSB経由でPCに接続
6. 以下のコマンドを実行（robot.portは上記で取得したシリアルポート名を使用）して、同様にID6～ID1の順に設定

    ```sh
    lerobot-setup-motors \
        --robot.type=so101_follower \
        --robot.port=/dev/tty.usbmodem5AB90672311
    ```

## 3. ロボットアームの組み立て

以下の点に注意して、<https://huggingface.co/docs/lerobot/so101> を参考にロボットアームを組み立てていく

- 3種類のネジがモーターに付属しているが、大きいものから順に、モーターの軸の中心に入れるネジ、モーターの軸とそれを挟み込むパーツを接続するネジ、モーターとモーターを囲うパーツを接続するネジとなっている
- 組み立てる際には、3ピンケーブルでモーター間をデイジーチェーン状に接続していく
  - モーターには2つの3ピンケーブル接続口があるが、モーター間はどのように接続してもよい
- モーター制御基盤に電源コードを繋ぐ際、**赤いコードをコンデンサー寄りの端子に繋ぐ**
  - ※内容の正確性は実機で要確認
- ロボットアームの3Dプリンタ製パーツは品質にばらつきがあり、ネジが奥まで入らないことがある。なるべくパーツを取り付ける**前**にネジが奥まで入るかを確認し、入らない場合はヤスリやリューターで削ってネジ穴を広げる
- 4番目のモーターの軸を挟み込むパーツを接続する前に、5番目のモーターに2つの3ピンケーブルを接続しておいたほうがよい
  - 4番目のモーターと5番目のモーターの接続をし、5番目のモーターの残りの1つの接続口にもケーブルをつないでおく
  - 5番目のモーターへの3ピンケーブルの接続が難しくなるため
- 5番目のモーターの軸に円盤状のパーツをはめるのは、5番目のモーターを囲うパーツにモーターを入れた後にする
  - 先にはめるとモーターを囲うパーツに入らなくなる

## 4. キャリブレーション（校正）

- ACアダプタをLeader用とFollower用で取り違えると、Leaderのモーターが一部認識しなくなるなど不安定な挙動になるため注意

以下の手順で、アームごとに関節（モーター）の可動範囲を計測し、リーダーアームとフォロワーアームの動作を同期できるようにする  
※以下、teleop.idやrobot.idには任意の名前（英数字）を指定できる。（複数ロボットを区別するためのIDで、キャリブレーションデータの保存ファイル名にも使われる）

1. リーダーアームをPCに接続し、以下のコマンドを実行（teleop.portは上記で取得したシリアルポート名を使用）

    ```sh
    lerobot-calibrate \
        --teleop.type=so101_leader \
        --teleop.port=/dev/tty.usbmodem5AB90678131 \
        --teleop.id=my_awesome_leader_arm
    ```

    以下のような表示がされる

    ```stdout
    Running calibration of my_awesome_leader_arm SO101Leader
    Move my_awesome_leader_arm SO101Leader to the middle of its range of motion and press ENTER....
    ```

2. すべての関節をそれぞれ中間の角度くらいになる姿勢に手で動かして、Enterキーを押す

    以下のような表示がされる

    ```stdout
    Move all joints sequentially through their entire ranges of motion.
    Recording positions. Press ENTER to stop...
    ```

3. 各関節を順番に端から端まで手で動かす

    以下のような表示がされる（各モーターの最小値(MIN)、現在値(POS)、最大値(MAX)がリアルタイムに更新表示される）

    ```stdout
    -------------------------------------------
    -------------------------------------------
    NAME            |    MIN |    POS |    MAX
    shoulder_pan    |    767 |   2325 |   3475
    shoulder_lift   |    731 |   1160 |   3327
    elbow_flex      |    886 |   3098 |   3099
    wrist_flex      |    877 |    883 |   3203
    wrist_roll      |    134 |    146 |   3884
    gripper         |   1971 |   1992 |   3190
    ```

4. すべての関節について最小～最大まで動かし終えたら、Enterキーを押す

    以下のような表示がされる（計測データがPC上のキャッシュ領域にJSONファイルとして保存される）

    ```stdout
    Calibration saved to /Users/<username>/.cache/huggingface/lerobot/calibration/teleoperators/so101_leader/my_awesome_leader_arm.json
    ```

5. 続いて、フォロワーアームをPCに接続し、以下のコマンドを実行（robot.portは上記で取得したシリアルポート名を使用）

    ```sh
    lerobot-calibrate \
        --robot.type=so101_follower \
        --robot.port=/dev/tty.usbmodem5AB90672311 \
        --robot.id=my_awesome_follower_arm
    ```

6. 後はリーダーアームと同様の手順を行う

    以下のような表示がされる（計測データがPC上のキャッシュ領域にJSONファイルとして保存される）

    ```stdout
    Calibration saved to /Users/<username>/.cache/huggingface/lerobot/calibration/robots/so101_follower/my_awesome_follower_arm.json
    ```

### 4.1. キャリブレーションデータ（計測データ）の実際の例

```sh
% cat ~/.cache/huggingface/lerobot/calibration/teleoperators/so101_leader/my_awesome_leader_arm.json
```

```json
{
    "shoulder_pan": {
        "id": 1,
        "drive_mode": 0,
        "homing_offset": -1747,
        "range_min": 767,
        "range_max": 3475
    },
    "shoulder_lift": {
        "id": 2,
        "drive_mode": 0,
        "homing_offset": -1455,
        "range_min": 731,
        "range_max": 3327
    },
    "elbow_flex": {
        "id": 3,
        "drive_mode": 0,
        "homing_offset": 328,
        "range_min": 886,
        "range_max": 3099
    },
    "wrist_flex": {
        "id": 4,
        "drive_mode": 0,
        "homing_offset": -1970,
        "range_min": 877,
        "range_max": 3203
    },
    "wrist_roll": {
        "id": 5,
        "drive_mode": 0,
        "homing_offset": 15,
        "range_min": 134,
        "range_max": 3884
    },
    "gripper": {
        "id": 6,
        "drive_mode": 0,
        "homing_offset": 1571,
        "range_min": 1971,
        "range_max": 3190
    }
}
```

```sh
% cat ~/.cache/huggingface/lerobot/calibration/robots/so101_follower/my_awesome_follower_arm.json
```

```json
{
    "shoulder_pan": {
        "id": 1,
        "drive_mode": 0,
        "homing_offset": -1653,
        "range_min": 740,
        "range_max": 3443
    },
    "shoulder_lift": {
        "id": 2,
        "drive_mode": 0,
        "homing_offset": -998,
        "range_min": 922,
        "range_max": 3320
    },
    "elbow_flex": {
        "id": 3,
        "drive_mode": 0,
        "homing_offset": 1253,
        "range_min": 746,
        "range_max": 2972
    },
    "wrist_flex": {
        "id": 4,
        "drive_mode": 0,
        "homing_offset": -1905,
        "range_min": 853,
        "range_max": 3229
    },
    "wrist_roll": {
        "id": 5,
        "drive_mode": 0,
        "homing_offset": 834,
        "range_min": 133,
        "range_max": 4049
    },
    "gripper": {
        "id": 6,
        "drive_mode": 0,
        "homing_offset": 1115,
        "range_min": 1847,
        "range_max": 3354
    }
}
```

## 5. テレオペレーションの実行（カメラなし）

以下のコマンドを実行すると、リーダーアームを操作してフォロワーアームが同じ動きを再現するテレオペレーションが行える  
※robot.idやteleop.idには、キャリブレーション時に設定したID名を入れる

```sh
lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem5AB90672311 \
    --robot.id=my_awesome_follower_arm \
    --teleop.type=so101_leader \
    --teleop.port=/dev/tty.usbmodem5AB90678131 \
    --teleop.id=my_awesome_leader_arm
```

## 6. カメラの検索

`lerobot-find-cameras`コマンドを実行すると以下のような表示がされ、outputs/captured_images ディレクトリにキャプチャされた画像が出力されるため、それによりカメラのindex番号を確認できる

```stdout
--- Detected Cameras ---
Camera #0:
  Name: OpenCV Camera @ 0
  Type: OpenCV
  Id: 0
  Backend api: AVFOUNDATION
  Default stream profile:
    Format: 16.0
    Fourcc: 
    Width: 1920
    Height: 1080
    Fps: 30.00003
--------------------
Camera #1:
  Name: OpenCV Camera @ 1
  Type: OpenCV
  Id: 1
  Backend api: AVFOUNDATION
  Default stream profile:
    Format: 16.0
    Fourcc: 
    Width: 1280
    Height: 800
    Fps: 10.0
--------------------
Camera #2:
  Name: OpenCV Camera @ 2
  Type: OpenCV
  Id: 2
  Backend api: AVFOUNDATION
  Default stream profile:
    Format: 16.0
    Fourcc: 
    Width: 1920
    Height: 1080
    Fps: 30.00003
```

なお、カメラをPCに接続する順番を変えてもindex番号は変わらなかった

## 7. テレオペレーションの実行（カメラあり）

カメラの情報を利用して以下のコマンドを実行するとRerun Viewerが開き、カメラ映像やアーム情報がリアルタイムに表示されつつテレオペレーションを実行できる

```sh
lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem5AB90672311 \
    --robot.id=my_awesome_follower_arm \
    --teleop.type=so101_leader \
    --teleop.port=/dev/tty.usbmodem5AB90678131 \
    --teleop.id=my_awesome_leader_arm \
    --robot.cameras="{side: {type: opencv, index_or_path: 2, width: 640, height: 360, fps: 30}, wrist: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 30}, top: {type: opencv, index_or_path: 1, width: 640, height: 360, fps: 30}}" \
    --display_data=true
```

## 8. 学習データの記録（Hugging Faceへのアップロードなし）

以下のコマンドを実行すると、学習データの記録が開始される。Macでは実行時にTerminalに矢印キーの許可をするかのダイアログが開く。許可してコマンドを再実行するとTerminalで矢印キーによる操作ができる。なお、この時点ではHugging Faceへのログインは不要

```sh
export HF_USER=＜Hugging Faceのユーザ名＞
lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem5AB90672311 \
    --robot.id=my_awesome_follower_arm \
    --robot.cameras="{side: {type: opencv, index_or_path: 2, width: 640, height: 360, fps: 30}, wrist: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 30}, top: {type: opencv, index_or_path: 1, width: 640, height: 360, fps: 30}}" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/tty.usbmodem5AB90678131 \
    --teleop.id=my_awesome_leader_arm \
    --display_data=true \
    --dataset.push_to_hub=False \
    --dataset.private=True \
    --dataset.repo_id=${HF_USER}/record-test \
    --dataset.single_task="Move the red cube into the black square area."  \
    --dataset.episode_time_s=60 \
    --dataset.reset_time_s=30 \
    --dataset.num_episodes=3 \
    --dataset.fps=30 \
    --dataset.video=true \
    --play_sounds=true
```

- 「Recording episode 0」のような音声が流れたら、`--dataset.episode_time_s`の指定時間（秒）以内にリーダーアームで学習したい操作を行って、時間を待つか「右矢印」キーで1つのエピソードが終了する
- 「Reset the environment」のような音声が流れたら、`--dataset.reset_time_s`の指定時間（秒）以内に（次の学習用に）環境をもとに戻し、時間を待つか「右矢印」キーで次のエピソードに進む
- 操作を失敗して現在のエピソードをやり直したい場合、「左矢印」キーで現在のエピソードをやり直すことができる（環境をもとに戻してからやり直せる）
- 途中で学習データの記録をやめたい場合、「ESC」キーで現在のエピソードを終了しつつ、学習データの記録も終了する
  - 現在のエピソードを記録せずに終了するには「左矢印」キーの後に「ESC」キーを押して終了する

## 9. 学習データの記録（Hugging Faceへのアップロードあり）

まずはHugging Faceでトークンを作成する必要がある。  
Hugging FaceのUser Access Tokensで、Repositoriesの`Write access to contents/settings of all repos under your personal namespace`
を許可してトークンを作成しておく。

`hf auth login`コマンドを実行し、トークンの入力を行ってログインする

```stdout
Enter your token (input will not be visible): 
Add token as git credential? (Y/n) Y
```

以下のコマンドのように、`--dataset.push_to_hub=True`と`--dataset.private=True`を指定して実行すると、コマンドの終了時にHugging Faceにprivateデータセットとして学習データがアップロードされる

```sh
export REPO_ID=＜データセット名＞
lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem5AB90672311 \
    --robot.id=my_awesome_follower_arm \
    --robot.cameras="{side: {type: opencv, index_or_path: 2, width: 640, height: 360, fps: 30}, wrist: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 30}, top: {type: opencv, index_or_path: 1, width: 640, height: 360, fps: 30}}" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/tty.usbmodem5AB90678131 \
    --teleop.id=my_awesome_leader_arm \
    --display_data=true \
    --dataset.push_to_hub=True \
    --dataset.private=True \
    --dataset.repo_id=${HF_USER}/${REPO_ID} \
    --dataset.single_task="Move the red cube into the black square area."  \
    --dataset.episode_time_s=60 \
    --dataset.reset_time_s=60 \
    --dataset.num_episodes=60 \
    --dataset.fps=30 \
    --dataset.video=true \
    --play_sounds=true
```

`--resume=true`オプションを指定すると同じデータセットに対して、`--dataset.num_episodes`の分だけさらに追加でエピソードを追加できる。ただし、既存データセットがHugging Faceに存在しないとエラーになる。

## 10. 学習の実施

### 10.1. 事前準備

- Hugging FaceとWandbでアカウントを作成しておく
- Hugging Faceでは、以下の権限を許可するトークンを作成して、これをログイン時のトークンとする
  - Repositories
    - `Read access to contents of all repos under your personal namespace`
    - `Read access to contents of all public gated repos you can access`
    - `Write access to contents/settings of all repos under your personal namespace`
- PaliGemma モデルの利用申請を <https://huggingface.co/google/paligemma-3b-pt-224> で行っておく
  - 画面で`Acknowledge license`ボタン → `Authorize`ボタン → `Accept`ボタン
  - 利用申請をしないと学習時に以下のエラーが発生する
  ```stdout
  403 Client Error: Forbidden for url: https://huggingface.co/google/paligemma-3b-pt-224/resolve/main/config.json
  ```

### 10.2. SageMaker AIのノートブックインスタンスで学習する場合の環境構築

※EC2での実施の方がおすすめ

事前にAWSのService Quotasの画面で`ml.p4d.24xlarge for notebook instance usage`のクォータを1以上に設定しておく

- ノートブックインスタンスのタイプ：`ml.p4d.24xlarge` インスタンス
  - `ml.g4dn.xlarge` インスタンスや`ml.g5.4xlarge` インスタンスではうまく動かなかった
- プラットフォーム識別子：`Amazon Linux 2023, Jupyter Lab 4`
- ボリュームサイズ (GB 単位)：200以上
  - pi0.5では1回の学習に最低20GB程度は使用する（20000ステップごとに20GB消費）　　

#### 10.2.1. Jupyter Notebook画面での操作

- 作成したノートブックインスタンスのステータスがInServiceになったらインスタンスの画面で「Jupyter を開く」を押して、Jupyter Notebook画面に入る
- Jupyter Notebook画面では、（通常のNotebookを作成するのではなく）New → Terminal を選択して、ターミナルを開いて作業する

#### 10.2.2. 環境構築手順

以下の手順で、conda仮想環境を作成して、仮想環境にactivateして入る

※`bash`コマンドを事前に実行していないと`CondaError: Run 'conda init' before 'conda activate'`のエラーが出る  
※`~/SageMaker`ディレクトリ以外は一時ディスク領域で、ノートブックインスタンスを停止した際に消去されるため、`~/SageMaker`ディレクトリ内に仮想環境を作成する

```sh
bash
conda create --prefix ~/SageMaker/lerobot_env -y python=3.10
conda activate ~/SageMaker/lerobot_env
```

ffmpegをインストール

```sh
conda install ffmpeg -c conda-forge
```

lerobotのリポジトリをclone  

```sh
cd ~/SageMaker
git clone https://github.com/huggingface/lerobot.git
cd lerobot
```

pipでライブラリをインストール  
※`No space left on device`のエラーを避けるため、`TMPDIR`を事前に指定

```sh
mkdir ~/tmp
export TMPDIR=~/tmp
pip install -e .
```

学習用のライブラリをインストール

```sh
bash
conda activate ~/SageMaker/lerobot_env
cd ~/SageMaker
pip install -e "./lerobot[pi]"
pip install -e "./lerobot[smolvla]"
```

以下のようなエラーが出るのを防ぐため

```stdout
I/O error: No space left on device (os error 28) at path "/home/ec2-user/.cache/huggingface/...
```

以下のようにシンボリックリンクを作成しておく

```sh
rm -rf ~/.cache/huggingface
mkdir -p /home/ec2-user/SageMaker/.cache/huggingface
ln -s /home/ec2-user/SageMaker/.cache/huggingface ~/.cache/huggingface
```

### 10.3. EC2インスタンスで学習する場合の環境構築

以下の設定でEC2インスタンスを作成し、セッションマネージャーで接続する

- AMI: `Deep Learning Base AMI with Single CUDA (Amazon Linux 2023)`
- インスタンスタイプ: `g6e.2xlarge`
- ルートボリューム: 200GB以上
- IAM インスタンスプロフィール: `AmazonSSMManagedInstanceCore`のポリシーを持つIAMロール

#### 10.3.1. AWS CLIとSession Manager Pluginのインストール（手元のPC）

接続元のPC（macOS）にAWS CLIとSession Manager Pluginをインストールしておく

```sh
brew install awscli
brew install --cask session-manager-plugin
```

インストール確認

```sh
aws --version
session-manager-plugin --version
```

#### 10.3.2. AWSプロファイルの設定

以下のコマンドでアクセスキーを使ったプロファイルを設定する  
※`<profile-name>` は任意の名前（例：`lerobot`）

```sh
aws configure --profile <profile-name>
```

入力する項目:

- AWS Access Key ID: `＜発行されたアクセスキーID＞`
- AWS Secret Access Key: `＜発行されたシークレットアクセスキー＞`
- Default region name: `ap-northeast-1`
- Default output format: `json`

設定内容は `~/.aws/credentials` と `~/.aws/config` に保存される。確認は以下のコマンドで行える

```sh
aws configure list --profile <profile-name>
```

#### 10.3.3. EC2インスタンスへのSession Manager接続

作成したEC2のインスタンスID（`i-xxxxxxxxxxxxxxxxx`）を控えておき、以下のコマンドで接続する

```sh
aws ssm start-session \
    --target i-xxxxxxxxxxxxxxxxx \
    --profile <profile-name>
```

接続直後は `ssm-user` でログインされているため、作業用ユーザに切り替えてからホームディレクトリに移動する  
※`<username>` は実際の作業用ユーザ名を指定

```sh
sudo su - <username>
cd ~
```

#### 10.3.4. 環境構築手順

Miniforgeをインストール（すべて、yesで回答）

```sh
bash
cd ~
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

`. .bashrc`を実行。または再ログインして`bash & cd ~`を実行

conda仮想環境を作成して、仮想環境にactivateして入り、ffmpegをインストール

```sh
conda create -y -n lerobot python=3.10
conda activate lerobot
conda install ffmpeg -c conda-forge
```

lerobotのリポジトリをcloneし、v0.4.2に切り替える

```sh
git clone https://github.com/huggingface/lerobot.git
cd lerobot
git checkout v0.4.2
cd ..
```

pipでライブラリをインストール  

```sh
cd lerobot
pip install -e .
pip install -e ".[feetech]"

cd ..
pip install -e "./lerobot[pi]"
pip install -e "./lerobot[smolvla]"
```

Pi0.5モデルの事前ダウンロード（v0.4.2を利用する場合）

lerobot v0.4.2では、Hugging Face Hub上の最新の`lerobot/pi05_base`がそのままでは利用できないため、事前に動作確認できているリビジョンを `/opt/hf_cache` にダウンロードしておく

```sh
sudo mkdir -p /opt/hf_cache
sudo HF_HOME=/opt/hf_cache HF_HUB_DISABLE_XET=1 /opt/miniforge3/envs/lerobot/bin/python -c "
from huggingface_hub import snapshot_download
p = snapshot_download('lerobot/pi05_base', revision='9e55186ad36e66b95cda57bc47818d9e6237ae30')
print('LOCAL_PATH:', p)
"
sudo chmod -R a+rX /opt/hf_cache
```

実行すると `LOCAL_PATH: /opt/hf_cache/hub/models--lerobot--pi05_base/snapshots/9e55186a.../` のようなパスが表示されるので、これをメモしておき、後述の学習実行時に `PI05_BASE_PATH_V042` 環境変数に指定する

### 10.4. 学習実行

Hugging FaceとWandbにログインする

```sh
hf auth login
wandb login
```

学習を実行する。wandbのサイトで、train/lossやtrain/grad_normの値が下がっていくことを確認する。
学習完了後は、wandbのサイトで（Freeアカウントの場合は5GBまでのため）必要に応じてArtifactsを削除しておく。

なお、`--dataset.image_transforms.enable=true`を指定すると拡張画像を利用して学習が行われる。※このオプションの使用により汎化性能が向上するか要確認

- Pi0.5での実行例

`PI05_BASE_PATH_V042` には、[10.3.4. 環境構築手順](#1034-環境構築手順)の事前ダウンロードで取得した `LOCAL_PATH` を指定する

```sh
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
export HF_USER=<Hugging Faceのユーザ名>
export PI05_BASE_PATH_V042=<事前ダウンロードで取得したLOCAL_PATH>

nohup lerobot-train \
  --dataset.repo_id=${HF_USER}/record-pick-place_20251210 \
  --dataset.revision=main \
  --dataset.video_backend=pyav \
  --policy.type=pi05 \
  --output_dir=./outputs/pi05_pick-place_20251217_50k_trans \
  --policy.repo_id=${HF_USER}/pi05_pick-place_20251217_50k_trans \
  --policy.pretrained_path=${PI05_BASE_PATH_V042} \
  --policy.compile_model=true \
  --policy.gradient_checkpointing=true \
  --policy.dtype=bfloat16 \
  --policy.push_to_hub=true \
  --policy.private=true \
  --steps=51200 \
  --policy.scheduler_decay_steps=51200 \
  --policy.device=cuda \
  --batch_size=2 \
  --num_workers=2 \
  --wandb.enable=true \
  --job_name=pi05_pick-place_20251217_50k_trans \
  > lerobot-train.log 2>&1 &
```

`ml.p4d.24xlarge` インスタンスでPi0.5モデルの学習の場合、200ステップの学習に約4分かかる  
`g6e.2xlarge` インスタンスでPi0.5モデルの学習の場合、200ステップの学習に約100秒かかる

- Pi0.5での実行例（Wandbのアップロードサイズを抑える場合）

`--wandb.disable_artifact=true` を指定するとWandbへのArtifactsアップロードが行われなくなり、Freeアカウントの5GB制限に引っかかりにくくなる。なお、このオプションはWandbへのアップロードのみをスキップするものであり、`--output_dir` 配下へのローカルチェックポイント保存には影響しないため、ローカルにはアーティファクト（チェックポイント）が引き続き保存される

```sh
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
export HF_USER=<Hugging Faceのユーザ名>
export PI05_BASE_PATH_V042=<事前ダウンロードで取得したLOCAL_PATH>

nohup lerobot-train \
  --dataset.repo_id=${HF_USER}/record-pick-place_20251210 \
  --dataset.revision=main \
  --dataset.video_backend=pyav \
  --policy.type=pi05 \
  --output_dir=./outputs/pi05_pick-place_20251217_50k_trans \
  --policy.repo_id=${HF_USER}/pi05_pick-place_20251217_50k_trans \
  --policy.pretrained_path=${PI05_BASE_PATH_V042} \
  --policy.compile_model=true \
  --policy.gradient_checkpointing=true \
  --policy.dtype=bfloat16 \
  --policy.push_to_hub=true \
  --policy.private=true \
  --steps=51200 \
  --policy.scheduler_decay_steps=51200 \
  --policy.device=cuda \
  --batch_size=2 \
  --num_workers=2 \
  --wandb.enable=true \
  --wandb.disable_artifact=true \
  --job_name=pi05_pick-place_20251217_50k_trans \
  > lerobot-train.log 2>&1 &
```

- SmolVLAでの実行例

```sh
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
export HF_USER=<Hugging Faceのユーザ名>

lerobot-train \
  --dataset.repo_id=${HF_USER}/record-pick-place_20251210 \
  --dataset.revision=main \
  --dataset.video_backend=pyav \
  --policy.type=smolvla \
  --output_dir=./outputs/smol_pick-place_20251222_50k \
  --policy.repo_id=${HF_USER}/smol_pick-place_20251222_50k \
  --policy.pretrained_path=lerobot/smolvla_base \
  --policy.push_to_hub=true \
  --policy.private=true \
  --steps=51200 \
  --policy.scheduler_decay_steps=51200 \
  --policy.device=cuda \
  --batch_size=2 \
  --num_workers=2 \
  --wandb.enable=true \
  --job_name=smol_pick-place_20251222_50k
```

`g6e.2xlarge` インスタンスでSmolVLAモデルの学習の場合、200ステップの学習に約30秒かかる

## 11. 推論の実施

lerobotのバージョンが学習時と異なるとエラーになることがあるため、同じバージョンであるか事前に確認する。異なるバージョンの場合、conda仮想環境の作成からやり直す。

- Pi0.5での実行例

lerobot 0.4.2でMacでの実行では`--policy.compile_model=false`を指定しないと`torch._inductor.exc.InductorError: NotImplementedError: welford_combine`のエラーが出る

```sh
lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem5AB90672311 \
    --robot.id=my_awesome_follower_arm \
    --robot.cameras="{side: {type: opencv, index_or_path: 2, width: 640, height: 360, fps: 30}, wrist: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 30}, top: {type: opencv, index_or_path: 1, width: 640, height: 360, fps: 30}}" \
    --dataset.repo_id=${HF_USER}/eval_${REPO_ID} \
    --dataset.single_task="Move the red cube into the black square area." \
    --dataset.num_episodes=1 \
    --dataset.episode_time_s=3000 \
    --dataset.reset_time_s=5 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false \
    --policy.path=${HF_USER}/${REPO_ID} \
    --policy.device=mps \
    --policy.compile_model=false \
    --display_data=true
```

- SmolVLAでの実行例

```sh
lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem5AB90672311 \
    --robot.id=my_awesome_follower_arm \
    --robot.cameras="{side: {type: opencv, index_or_path: 2, width: 640, height: 360, fps: 30}, wrist: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 30}, top: {type: opencv, index_or_path: 1, width: 640, height: 360, fps: 30}}" \
    --dataset.repo_id=${HF_USER}/eval_${REPO_ID} \
    --dataset.single_task="Move the red cube into the black square area." \
    --dataset.num_episodes=1 \
    --dataset.episode_time_s=3000 \
    --dataset.reset_time_s=5 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false \
    --policy.path=${HF_USER}/${REPO_ID} \
    --policy.device=mps \
    --display_data=true
```

`Ctrl-C`でコマンドを終了したとき、モーターのトルクが固定されてしまった場合はアームの電源を切って解除する
