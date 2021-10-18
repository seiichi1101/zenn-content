---
title: "Google OAuth2.0を使ってGoogle Tasks APIを呼びすReact SPAを作ってみた"
emoji: "🐫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react","typescript","oauth"]
published: true
---

Reactで作成したSPAクライアントからGoogleのOAuth2.0を利用してアクセストークンを取得し、GoogleのTasks APIを呼び出してみました。

## Google Tasks API ってなに？

[Google Tasks API](https://developers.google.com/tasks)は、Googleから提供されているタスク管理ツール「Google Tasks(グーグルタスク)」を呼び出せるAPIです。

「Google Tasks(グーグルタスク)」は簡単に言えばTODOリストの機能を提供してくれます。

「Google Tasks(グーグルタスク)」自体はユーザーに作成されたタスク情報を管理しており、アクセストークンさえあれば[モバイルアプリ](https://play.google.com/store/apps/details?id=com.google.android.apps.tasks&hl=en&gl=US)や[ウェブアプリ](https://tasksboard.com/app)など様々なクライアントからタスク情報を呼び出すことが可能です。

## どうやってアクセストークンを取得するのか？

[Googleから提供されているOAuth2.0フロー](https://developers.google.com/identity/protocols/oauth2)に沿ってGoogleのユーザー認証を行えばアクセストークンが取得できます。

フローは、クライアントの種別によって複数提供されています。

クライアントの種別に応じた適切なフローを利用しないと、セキュリティリスクがあるので注意が必要です。

今回は、React SPAのパブリッククライアントということで、client_idのみで利用できる[OAuth 2.0 for Client-side Web Applications](https://developers.google.com/identity/protocols/oauth2/javascript-implicit-flow)に沿って実施していきます。

フローはOAuth2.0の**Implicit Grant Flow**になります。

![](https://developers.google.com/identity/protocols/images/oauth2/device/flow.png)

## やってみた

## Google Cloudの設定

まずGoogle Cloud側で２点ほどやることがあります。

#### 1.クライアントアプリの登録

[認証情報](https://console.cloud.google.com/apis/credentials)ページに遷移します

- 「認証情報作成」 -> 「OAuth クライアント ID」 をクリックします

![](/images/google-oauth2-react-spa/2021-10-18_14h23_25.png)

- Webアプリケーションのアプリケーションタイプを選択し必要な項目を入力します

![](/images/google-oauth2-react-spa/2021-10-18_14h25_57.png)

ClientIdは後で必要になるのでメモしておきます。

#### 2.Google Tasks APIの有効化

あとでこのAPIを使うので有効化しておきます。

コンソールの検索Boxで「tasks api」と入力すれば候補に表示されるはずです。

![](/images/google-oauth2-react-spa/2021-10-18_14h28_19.png)

## React SPAのセットアップ

- プロジェクトの作成

`npx create-react-app my-google-tasks-spa --template typescript`

- ライブラリのインストール

`npm i react-google-login @mui/material @emotion/react @emotion/styled axios`

あとから使うやつをまとめてインストールしておきます。

## Googleでのユーザー認証の実装

[公式ドキュメント](https://developers.google.com/identity/protocols/oauth2/javascript-implicit-flow)によると、

1. JS Client Library
2. OAuth 2.0 Endpoints

の２つのパターンが用意されています。

1はGoogleが提供している公式のJSライブラリ[Google API Client Library for JavaScript](https://github.com/google/google-api-javascript-client)を使う方法、2はImplict Flowで必要となるサーバーとのやり取りを自前で実装する方法です。

1の方法でやろうと思ったのですが、このライブラリがCDNで提供されており、Reactで利用するには少し手間だったので、今回は3rd Partyライブラリ[React Google Login](https://github.com/anthonyjgrove/react-google-login)を使うことにします。

- `.env`ファイル

```
REACT_APP_CLIENT_ID=<your-client-id>
```

- サンプルコード `APP.tsx`

```tsx:APP.tsx
import React, { useState } from 'react';
import './App.css';
import { GoogleLogin } from 'react-google-login';

const ClientId = process.env.REACT_APP_CLIENT_ID!;

function App() {
  const [accessToken, setAccessToken] = useState<string>('');

  const onSuccess = (
    res: ReactGoogleLogin.GoogleLoginResponse | ReactGoogleLogin.GoogleLoginResponseOffline
  ) => {
    if ('accessToken' in res) {
      console.log(res.accessToken);
      setAccessToken(res.accessToken);
    }
  };
  const onFailure = (res: any) => {
    alert(JSON.stringify(res));
  };

  return (
    <div className="App">
      {accessToken === '' ? (
        <GoogleLogin
          clientId={ClientId}
          buttonText="Login"
          onSuccess={onSuccess}
          onFailure={onFailure}
          scope="https://www.googleapis.com/auth/tasks"
          cookiePolicy={'single_host_origin'}
        />
      ) : (
        <>{accessToken}</>
      )}
    </div>
  )
}

export default App;
```

- 確認

`npm start` で確認してみると分かると思いますが、シンプルなログインボタンが追加されます。

ログインボタンをクリックするとOAuth2.0のフローが開始されます。

アクセス許可を求められると思いますが、承認します。

すべて完了すれば画面にはアクセストークンが表示されるはずです。

![](/images/google-oauth2-react-spa/2021-10-18_15h11_25.png)

## Google Tasks APIの呼び出し

無事アクセストークンが取得できたので、このトークンを使ってGoogle Tasks APIを呼び出してみたいと思います。

[API Reference](https://developers.google.com/tasks/reference/rest)を見るといろいろ昨日はありそうですが、今回はシンプルにタスクの`Create`と`Delete`のAPIを呼び出す実装をします。

また、Google Tasksではタスクリスト(`TaskList`)という単位で複数のタスクを１つにまとめているのですが、今回はデフォルトで作成されている最初のタスクリストのみを利用します。

- サンプルコード `APP.tsx`

※あまりTypeScriptに慣れていないので、あくまでサンプル程度に考えて頂けたらと思います。

```diff tsx:APP.tsx
-import React, { useState } from 'react';
+import React, { useEffect, useState } from 'react';
 import './App.css';
+import Button from '@mui/material/Button';
+import Stack from '@mui/material/Stack';
+import List from '@mui/material/List';
+import ListItem from '@mui/material/ListItem';
+import axios from 'axios';
 import { GoogleLogin } from 'react-google-login';
 
+const BaseUrl = 'https://tasks.googleapis.com';
 const ClientId = process.env.REACT_APP_CLIENT_ID!;
 
-function App() {
+type TaskList = {
+  id: string;
+  title: string;
+  tasks: Task[];
+};
+
+type Task = {
+  id: string;
+  name: string;
+};
+
+const App: React.FC = () => {
   const [accessToken, setAccessToken] = useState<string>('');
+  const [taskList, setTaskList] = useState<TaskList>();
+  useEffect(() => {
+    const setFirstTaskList = async (accessToken: string) => {
+      const firstItem = await loadFirstTaskList(accessToken);
+      const tasks = await loadTasks(firstItem.id);
+      setTaskList({ id: firstItem.id, title: firstItem.title, tasks } as TaskList);
+    };
+    if (accessToken) {
+      setFirstTaskList(accessToken);
+    }
+  },[accessToken]);
 
   const onSuccess = (
     res: ReactGoogleLogin.GoogleLoginResponse | ReactGoogleLogin.GoogleLoginResponseOffline
   ) => {
     if ('accessToken' in res) {
       console.log(res.accessToken);
       setAccessToken(res.accessToken);
     }
   };
   const onFailure = (res: any) => {
     alert(JSON.stringify(res));
   };
 
+  const loadFirstTaskList = async (accessToken: string) => {
+    const res = await axios.get<any>(`${BaseUrl}/tasks/v1/users/@me/lists`, {
+      headers: { Authorization: `Bearer ${accessToken}` },
+    });
+    return res.data.items[0];
+  };
+  const loadTasks = async (id: string) => {
+    const res = await axios.get<any>(`${BaseUrl}/tasks/v1/lists/${id}/tasks`, {
+      headers: { Authorization: `Bearer ${accessToken}` },
+    });
+    if (!res.data.items) {
+      return [];
+    }
+    return await res.data.items.map((t: any) => {
+      return { id: t.id, name: t.title } as Task;
+    });
+  };
+
+  const reload = async () => {
+    if (!taskList) {
+      return;
+    }
+    const tasks = await loadTasks(taskList.id);
+    setTaskList({ id: taskList.id, title: taskList.title, tasks } as TaskList);
+  };
+  const create = async () => {
+    if (!taskList) {
+      return;
+    }
+    await axios.post(
+      `${BaseUrl}/tasks/v1/lists/${taskList.id}/tasks`,
+      { title: `task${taskList.tasks.length + 1}` },
+      { headers: { Authorization: `Bearer ${accessToken}` } }
+    );
+    reload();
+  };
+  const remove = async () => {
+    if (!taskList) {
+      return;
+    }
+    await axios.delete(`${BaseUrl}/tasks/v1/lists/${taskList.id}/tasks/${taskList.tasks[0].id}`, {
+      headers: { Authorization: `Bearer ${accessToken}` },
+    });
+    reload();
+  };
+
   return (
     <div className="App">
       {accessToken === '' ? (
         <GoogleLogin
           clientId={ClientId}
           buttonText="Login"
           onSuccess={onSuccess}
           onFailure={onFailure}
           scope="https://www.googleapis.com/auth/tasks"
           cookiePolicy={'single_host_origin'}
         />
       ) : (
-        <>{accessToken}</>
+        <>
+          <h2>{taskList?.title}</h2>
+          <Stack direction="row" spacing={2} alignItems="center" justifyContent="center">
+            <Button variant="contained" onClick={create}>
+              create
+            </Button>
+            <Button variant="contained" onClick={remove}>
+              delete
+            </Button>
+          </Stack>
+          <Stack spacing={2} alignItems="center" justifyContent="center">
+            <List>
+              {taskList?.tasks?.map((e) => (
+                <ListItem key={e.id}>{e.name}</ListItem>
+              ))}
+            </List>
+          </Stack>
+        </>
       )}
     </div>
-  )
-}
+  );
+};
 
 export default App;
```

- 確認①

先ほどと同様に`npm start`で動作確認しています。

![](/images/google-oauth2-react-spa/2021-10-18_17h48_59.png)

- 確認②

[TasksBoard](https://tasksboard.com/app)というウェブアプリケーションに同じGoogleアカウントでログインすると、同じタスクが登録されているのが確認できるはずです。

![](/images/google-oauth2-react-spa/2021-10-18_17h55_51.png)

## まとめ

今回はパブリッククライアントから利用できるOAuth2.0に対応しているサービスを探している中で、Googleを試してみました。

GitHubも試してみましたが、[こちら](https://docs.github.com/en/developers/apps/building-oauth-apps/authorizing-oauth-apps)にある通り認証フローの中で`client_secret`を送信する必要があり断念しました。

Twitterは現在ベータ版でOAuth2.0をリリースしており、ベータプログラムに参加すれば試せるそうです。
[こちら](https://zenn.dev/kg0r0/articles/8d787860e9b2e1)の記事を参考にする限り`client_secret`の送信は必要なさそうです。リリースされたら試して見ようと思います。

あまり情報がないため、意外とはまりどころの多かったです。

どなたかの役に立てば幸いです。

## あとがき

- Google認証の「OAuth 2.0 クライアント ID」は一度あるポートで使うと、その後で違うポートで利用することはできないらしいので、変更したい場合は新しくクライアントを登録する必要があるとのこと
  - <https://qiita.com/kenken1981/items/9d738687c5cfb453be19>
- [tasks.update](https://developers.google.com/tasks/reference/rest/v1/tasks/update)のAPIがうまく動かなかった
  - `title`を変更して、200 OKまで返ってくるが`tasks.list`を実行しても変化がない
  - 一方で、`tasks.insert`を実行して新しいタスクを作成したら、過去の変更が反映された
