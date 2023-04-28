---
title: "GPTに自分が読んでいる記事の内容を読んでもらった"
emoji: "🤟"
type: "tech"
topics:
  - "angular"
  - "gpt"
published: true
published_at: "2023-04-06 15:18"
publication_name: "chot"
---

初めまして
chot Inc.でウェブフロントエンジニアをしているぴです

GPT に自分が今読んでいる記事の内容を読んでもらって、それについて会話できたら最強かもと思ったので、デモアプリとして Chrome 拡張を作ってみました。

## 作ったもの

個人ブログのページを読んでもらって、追加で質問をした時のスクショです。
GPT-3.5-turbo を使っているのですが、かなり使えそうです。
![](https://storage.googleapis.com/zenn-user-upload/b14c53d4e129-20230406.png)

### リポジトリ

https://github.com/2ndPINEW/popup-reading-gpt

試してみたい方は、クローンして試してみてください。

試す際の注意点として、最初に読み込んだ時に API キーの入力を求められるのですが、保存先が`chrome.storage.local`です。
[ドキュメント](https://developer.chrome.com/docs/extensions/reference/storage/#storage-areas)に機密情報入れるなって書いてあったのですが、`session`に入れるとブラウザ開き直すたびに API キー入れなきゃダメで、どうすればいいかわからなかったです。

## 実装方法

拡張機能の動作の流れとしては

- 開いているページの本文を読み込む
- ChatGPT の API を本文とプロンプトを渡して叩く
- 追加の質問があればメッセージに追加して API を叩く

この機能を実装していきます。
API 叩く部分は fetch しているだけなので、開いているページの本文を読み込む部分を主に解説していきます。

フロントエンドのフレームワークに Angular 使ってるけど許してください、、、、、、、、、
ルーターも使ってなくて、ほぼ状態もないので本当に必要なかった、むしろ無駄に Rx するせいで読みづらくなった、、、

### 開いているページの本文を読み込む

HTML から記事本文を抜き出すのには以下のパッケージを利用しました
https://github.com/extractus/article-extractor

また、開いているページの html を読み取るために [chrome.scripting](https://developer.chrome.com/docs/extensions/reference/scripting/) API を使用しました

以下が、開いているページの本文を読み込むためのサービスです

```typescript
import { Injectable } from "@angular/core";
import { from, map, Observable, of, switchMap } from "rxjs";
import { ArticleData, extractFromHtml } from "@extractus/article-extractor";
import { environment } from "src/environments/environment";

export declare const chrome: any;

@Injectable({
  providedIn: "root",
})
export class ClientContentService {
  // ページに表示されている記事コンテンツを取得する関数
  getArticle$(): Observable<ArticleData | null> {
    if (!environment.production) {
      // 開発環境ではサンプルデータを返す
      return of({ content: "This is sample content" });
    }

    // 開いているタブの記事データを返すObservable
    return from(this.getTabInfo()).pipe(
      switchMap(({ tabId, url }) =>
        this.getTabHtml$(tabId).pipe(
          switchMap((html) => this.extractArticleData(html, url))
        )
      )
    );
  }

  // 現在開いているタブの情報を取得する関数
  private async getTabInfo(): Promise<{ tabId: number; url: string }> {
    const tabs = await chrome.tabs.query({
      active: true,
      currentWindow: true,
    });
    return { tabId: tabs[0].id, url: tabs[0].url };
  }

  // 指定したタブのHTMLを取得する関数
  private getTabHtml$(tabId: number): Observable<string> {
    return from(
      chrome.scripting.executeScript({
        target: { tabId },
        func: () => {
          {
            return document.querySelector("html")?.outerHTML;
          }
        },
      })
    ).pipe(map((injectionResults: any) => injectionResults[0].result));
  }

  // HTMLから記事データを取得する
  private async extractArticleData(
    html: string,
    url: string
  ): Promise<ArticleData | null> {
    const article = await extractFromHtml(html, url);
    if (article?.content) {
      article.content = article?.content?.replace(/(<[^>]+>|\{[^}]+\})/g, "");
    }
    return article;
  }
}
```

`getArticle$()`が呼ばれた時は、`getTabInfo()`で開いているタブの ID と url を取得します。そのあと`getTabHtml$()`で`html`を取得します。最後に`extractArticleData()`で`html`と`url`から記事のデータを取り出す流れになっています。

これで`getArticle$()`関数を使用して、記事の本文を文字列で取得することができるようになりました。
記事の本文が取得できたら、あとは ChatGPT の API に内容を渡すだけです！

### ChatGPT の API を本文とプロンプトを渡して叩く

次に先ほど取得した本文とプロンプトを渡して API を叩きます。最初に API に渡すメッセージは以下のようにしました。

```typescript
...略
private initMessages(articleContent: string) {
  this.messages = [
    {
      role: 'system',
      content:
        'あなたは優秀なアシスタントです、以下の文章の内容をよく読んで、要点を要約してください。また日本語で回答してください。ユーザーからの質問に回答する際は、文章内にある情報が有用な場合、文章内の情報を優先して利用してください',
    },
    {
      role: 'system',
      content: articleContent,
    },
  ];
}
...略
```

メッセージを初期化した後に API を叩いて、その応答をユーザーに表示します。

### 追加の質問を受け付ける

追加の質問があった際には、messages 変数に

```typescript
{
  role: 'user',
  content: `${質問内容}`,
}
```

質問内容のメッセージを追加して、再度 API を叩くようにしました。

API を叩く部分などは、サービスでファイルが分かれていて記事に貼りづらかったので、リポジトリを見ていただけると助かります。

## 最後に

ChatGPT に聞くと技術的に古い回答が返ってきてしまうことがあるのですが、新しい情報を読ませた上で、その話をできたのでとても良い体験ができました。
次に気になってるもので、LangChain の Agent から　 Google 検索とかさせたらこの拡張も必要なくなるのかなって気がしてワクワクしてるので、次は LangChain とおしゃべりできたらいいなと思っています。
