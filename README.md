# first-bolt-app

## はじめに

Slack の Appを、Slack の公式フレームワーク Bolt で作ってみようと思いチュートリアル https://slack.dev/bolt-js/tutorial/getting-started をやってみました。

英語版です。

チュートリアルには日本語版もありますが英語版でやってみることをお勧めします。いずれ日本語版のチュートリアルも英語版と同じ内容に更新されるとよいのですが。

* 英語版は Socket Mode を使っていて始めやすい
  * HTTPモードだとインターネット上にエンドポイントが必要
  * Socket Modeだとファイアウォールの中でも動かせる
* 日本語版は最新の UI との差異が多い
* 内容も英語版の方が実用的かつ充実
  * 日本語版は App Home を参照したときの処理を実装するが、英語版はメッセージが投稿されたときの処理を実装する
  * 英語版ではボタンの表示・ボタンが押されたときの処理までカバー

私はJavaScript版でやりましたがPython版、Java版もあります。

以下はあらすじです。

## 前提

始める前に以下を確認しておいてください。

* 管理者権限があって練習用に使えるワークスペースがあること
  * 無料で作成でき、Slackも推奨している
* Node.js が使える環境であること

私は Windows 11 + WSL2(Ubuntu) でやりました。vscode + Remote拡張だとエディタはいつも通り使えて、かつターミナルの中は正真正銘の Linux なので面倒がありません。

## 注意点

以下にご注意ください。

* GitHub を使っているのは自分がそうやったというだけで、チュートリアルにそう書いてあるわけではありません。
* `.env` や `package.json` についてもチュートリアルに書いてないことをやっています。

## App の作成

まずは Slack 側で App を作成します。

App は https://api.slack.com/apps/new で "From Scratch" から作成します。 

作成すると App 管理画面の Basic Information ページが表示されます。App 管理画面はこの後も利用します。後で戻ってきたいときは https://api.slack.com/apps から。

次に、アプリに許可する操作を選びます。

ここでは、OAuth & Permissions メニューの Scopes で `chat:write` を追加します。

最後に Install to Workspace で App をインストールします。インストールするとBot User OAuth Token が表示されます。これは後で使いますが後から確認できますのでメモっておく必要はありません。

これでワークスペースのサイドバーに App が表示されるようになります。

## ローカル環境の準備

まずは作業フォルダを作成します。自分は GitHub でリポジトリを作ってクローンしました。Node 用の `.gitignore` を追加しておきたかったりいろいろ。

作業フォルダに移動したら `npm init` します。`app.js` という名前でファイルを作るので、`entry point` には `app.js` を指定しておきます。

## 環境変数の設定

環境変数 `SLACK_SIGNING_SECRET` と `SLACK_BOT_TOKEN` を設定します。

`SLACK_SIGNING_SECRET` は Basic Information メニューの Signing Secret を、`SLACK_BOT_TOKEN` は OAuth & Permissions メニューで Bot User OAuth Token を確認します。

私は `.env` ファイルを作って `. .env` で反映しました。

```bash
export SLACK_SIGNING_SECRET=<Signing Secret>
export SLACK_BOT_TOKEN=<Bot User OAuth Token>
```

`.gitignore` に `.env` を追加しておきます。

## Bolt のインストール

Bolt のインストールは `npm install` するだけ。

```shell
$ npm install @slack/bolt
```

## 最小の App 作成

`app.js` ファイルを作成してコードを入力します。

```javascript
const { App } = require('@slack/bolt');

const app = new App({
  token: process.env.SLACK_BOT_TOKEN,
  signingSecret: process.env.SLACK_SIGNING_SECRET
});

(async () => {
  await app.start(process.env.PORT || 3000);
  console.log('⚡️ Bolt app is running!');
})();
```

実行します。今は何もしないのでエラーが出なければ OK。

```shell
$ node app.js
⚡️ Bolt app is running!
```

Ctrl-Cで止まります。

`npm start` で起動できるように `package.json` に起動スクリプトを追加しておきます。

```json
  :
  "scripts": {
    "start": "node app.js",
  :
```

## Socket Mode を有効にする

Socket Mode にすると、エンドポイントをインターネットに露出しなくてもイベントの受信が可能になります。ファイアウォールや NAT の内側でも App を動かしておけるのでお手軽。

App 管理画面の Socket Mode メニューで Enable Socket Mode をオンにし、次の画面で Token Name に Socket Mode などと入れて Generate します。

App-level Tokenが発行されます。後で見たいときは Basic Information メニューで。

## Event を Subscribe する

App 管理画面の Event Subscriptions メニューで Enable Events を On にして、Subscribe to bot events で message.channels を追加します。
これはパブリックなチャンネルでメッセージが送られたというイベントで、ほかにプライベートチャンネルやDM用のイベントもあります。

Save Changes を忘れないように。

権限が変わるのでアプリを再インストールしろと警告が出るのでreinstall your app をクリックします。

環境変数に App-level Token を追加して `. .env` で反映。

```bash
export SLACK_SIGNING_SECRET=<Signing Secret>
export SLACK_BOT_TOKEN=<Bot User OAuth Token>
export SLACK_APP_TOKEN=<App-level Token>
```

## Socket Mode での起動

App オブジェクトで Socket Mode を指定します。

```javascript
const app = new App({
  token: process.env.SLACK_BOT_TOKEN,
  signingSecret: process.env.SLACK_SIGNING_SECRET,
  socketMode: true,  // ここ
  appToken: process.env.SLACK_APP_TOKEN // ここ
  port: process.env.PORT || 3000  // ここ Socket Modeでもつけておいたほうがよい
});
```

## イベントハンドラを追加

メッセージが投稿された時のハンドラは `app.message` で登録します。

```javascript
const app = new App({
  :
});

// 追加
app.message('hello', async ({ message, say }) => { 
  await say(`Hey there <@${message.user}>!`);
});
```

`say` の引数の中の `<@...>` はメンションに置き換えられて表示されます。

## 実行

App を実行してSlackのパブリックチャンネルにボットを `/invite` したら `hello` と送信します。

"Hey there @<ユーザー名>!" と表示されたら成功！ここまででも最低限のボットは作れそうです。

## 返答にボタンを追加する

`say` にブロックを渡してやるとリッチなメッセージを送れます。ここではボタンを追加。

```javascript
app.message('hello', async ({ message, say }) => {
  await say({
    blocks: [
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `Hey there <@${message.user}>!`
        },
        accessory: {
          type: 'button',
          text: {
            type: 'plain_text',
            text: 'Click Me'
          },
          action_id: 'button_click'
        }
      }
    ],
    text: `Hey there <@${message.user}>!`
  });
});
```

ブロックでいろいろ試すなら https://app.slack.com/block-kit-builder から。

## ボタンのハンドラを追加する

ボタンが押されたときのイベントを処理するハンドラを追加します。

```javascript
app.action('button_click', async ({ body, ack, say }) => {
  await ack();
  await say(`<@${body.user.id}> clicked the button`);
});
```

"Click Me"をクリックするとメッセージが表示されます。

チュートリアルはここでおしまい。あとは次に読むべきドキュメントの紹介など。

## 気づき1

このままだとメッセージの中に "hello" が含まれていれば何でも呼ばれてしまいます。"othello" とかでも。

呼ばれてからチェックしてもいいんですが、正規表現でひっかけることも可能です。

```javascript
app.message(/^hello/, async ({ message, say }) => {
```

メンションに反応するイベントを使うとか、スラッシュコマンドにするとかの方が素直かもしれません。

## 気づき2

エラーが発生すると止まってしまいます。

```javascript
app.message('hello', async ({ message, say }) => {
  await say(`Hey there <@${message.user}>!`);
  throw new Error('oh');  // appが止まってしまう 
});
```

hubot は（エラーを無視して）勝手に続けてくれてたんですが、Bolt では自分で処理してやらないといけません。

```javascript
app.message('hello', async ({ message, say }) => {
  try {
    await say(`Hey there <@${message.user}>!`);
    throw new Error('oh');
  } catch (error) {
    console.log(error);
  }
});
```

appが止まったら起動するようなしくみを外付けする手もありそう。
