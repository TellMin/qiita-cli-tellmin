---
title: C#でChatGPT APIを触りたい with Betalgo.OpenAI.GPT3
tags:
  - C#
  - 入門
  - Blazor
  - ChatGPT
private: false
updated_at: '2023-06-14T23:13:36+09:00'
id: 7baaba35111fddeffe0c
organization_url_name: null
slide: false
ignorePublish: false
---
[6/14 追記]

`Betalgo.OpenAI.GPT3`は非推奨になっており、代わりに[`Betalgo.OpenAI`](https://github.com/betalgo/openai)が利用可能です。
当記事の内容については、名前空間を修正することで動作します。

新しく記事を書きました！こちらも是非！
:arrow_forward: [C#でChatGPT APIを触りたい 2 ~ ReadableStream ~](https://qiita.com/TellMin/items/9059423600a6897cef0c)

---

## この記事は？ :dango:

- ChatGPT APIをC#で叩くよ
- [Betalgo.OpenAI.GPT3](https://github.com/betalgo/openai) v6.7.0 を利用するよ
- 成果物はこちら [TellMin/ChatGPT_Blazor](https://github.com/TellMin/ChatGPT_Blazor)

## 対象読者 :dark_sunglasses:

- C#でChatGPT APIを叩きたい人へ

## はじめに :cactus: 

ChatGPTが提供されてからQiitaにたくさんの記事が投稿されていますが、フロントエンド界隈と違い、C#のChatGPT APIライブラリを紹介している記事がないため、本記事を書きました。

## 基本実装 :zap: 

基本の実装は [Readme](https://github.com/betalgo/openai/blob/master/Readme.md) にて詳しく紹介されていますが、ざっと下記のように実装します。

```C#
var completionResult = 
    await openAIService.ChatCompletion.CreateCompletion
    (
        new ChatCompletionCreateRequest()
        {
            Messages = chatMessages,
            Model = Models.ChatGpt3_5Turbo
        }
    );

if (completionResult.Successful) return completionResult;

```

[ChatService.cs](https://github.com/TellMin/ChatGPT_Blazor/blob/a69f5042da34daa30449db9eb201fee02a278d03/ChatGPT_Blazor/Services/ChatService.cs#L21-L34)

APIのモデルに関しては構造自体は変わらないため、Messagesに会話内容を詰め込めばAIの回答が返ってきます。

### セットアップ :secret: 

APIを叩くためにはOpenAIのAPIキーを渡す必要があります。
`Betalgo.OpenAI.GPT3`では直接キーを渡して生成する方法と`secrets.json`からの自動取得両方に対応しています。[^d]

[^d]: `Betalgo.OpenAI.GPT3`に限った話ではないですが、APIキーやToken情報を`appsettings.json`などのGit管理下に置かないよう気を付けましょう

```C#

// サービスを生成する場合
var openAiService = new OpenAIService(new OpenAiOptions()
{
    ApiKey =  Environment.GetEnvironmentVariable("MY_OPEN_AI_API_KEY")!, required
    Organization = Environment.GetEnvironmentVariable("MY_OPEN_ORGANIZATION_ID") //optional
});

// DIする場合は自動でセットしてくれる
serviceCollection.AddOpenAIService();

```

## 応用編 :fire: 

ここからは、どのようなpromptを渡して活用するかの話になります。自身の開発事例を紹介します。

### 成果物

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2566826/b1e68a06-1b9d-a896-e92d-71669efc7237.png)

上記ページではAIが英語学習の先生となり、ユーザーの作文を添削しながら会話を続けています。
このような応答を生成するために、今回は次のようなプロンプトを用意しました。

```C#

    private List<ChatMessage> chatMessages { get; set; } = new List<ChatMessage>()
    {
        ChatMessage.FromSystem("You are a strict teacher of English grammar and spelling errors."),
        ChatMessage.FromSystem("We are in the middle of an English lesson and we practice daily conversation."),
        ChatMessage.FromSystem("Be sure to output the data according to the following format. Correct: <YOUR_CORRECTION> CorrectReason: <YOUR_CORECTION_REASON> Respond: <YOUR_RESPOND>"),
        ChatMessage.FromSystem("In the \"Correct\" part, correct any grammatical or spelling errors, suggest better wording, and respond to the message with your reasons in 'CorrectReason' part."),
        ChatMessage.FromSystem("In 'Respond' part, act as an American who likes conversation and respond message."),
    };

```

そして、返ってきた応答を次のメソッドで

- Assistentのメッセージ
- 校正後の全文
- 校正理由

の3つのセクションに分割しています。

```C#

    private List<string> SplitMessage(string contets)
    {
        var split = contets.Split("CorrectReason:");
        var correct = split[0].Replace("Correct:", "").Trim();
        var correctReason = split[1].Split("Respond:")[0].Trim();
        var respond = split[1].Split("Respond:")[1].Trim();
        var result = new List<string>();

        result.Add(correct ?? string.Empty);
        result.Add(correctReason ?? string.Empty);
        result.Add(respond ?? string.Empty);

        return result;
    }

```

> ちなみにこちらはGithub Copilotに生成してもらいました。
> 大変便利な世の中になったものです。

今回の記事では特に紹介しませんが、今回の開発では次の技術も採用しました。

- [Radzen.Blazor](https://github.com/radzenhq/radzen-blazor)
- .NET 7 × Visual Studioでコンテナ開発
- Visual StudioからECSへのデプロイ

これでこの記事の全コンテンツが終了です。
ちなみに、この記事は ChatGPT によって校正されました。
AIの活躍がますます期待されていますね。
C#の明日はどうなるのでしょうか？

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2566826/4ae2cd29-2271-ffbe-eda0-3738c8094f54.png)
