---
title: C#でChatGPT APIを触りたい 2 ~ ReadableStream ~
tags:
  - C#
  - 入門
  - Blazor
  - ChatGPT
private: false
updated_at: '2023-06-15T06:11:38+09:00'
id: 9059423600a6897cef0c
organization_url_name: null
slide: false
---
## この記事は :apple: 

- ChatGPT APIをC#で叩くよ
- [Betalgo.OpenAI](https://github.com/betalgo/openai) v7.0.1 を利用するよ
- 成果物はこちら [TellMin/ChatGPT_Blazor - stream](https://github.com/TellMin/ChatGPT_Blazor/tree/stream)

:small_red_triangle_down:結果
![ChatStream.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2566826/06f96d5b-3595-b44c-5dae-ba89911c6156.gif)

## 対象読者 :whale:

- C#でChatGPT APIを叩きたい人へ
- ChatGPTの画面のように、少しずつレスポンスを返したい方へ

## 初めに :seedling:

以前、 [`C#でChatGPT APIを触りたい with Betalgo.OpenAI.GPT3`](https://qiita.com/TellMin/items/7baaba35111fddeffe0c)という記事の中で`Betalgo.OpenAI.GPT3`の利用方法について紹介しました。
時がたち、名前空間や機能に変更がありましたので、試してみました。

## 実装 :star:

レスポンスを[`ReadableStream - Web API|MDN`](https://developer.mozilla.org/ja/docs/Web/API/ReadableStream)として扱ってくれる、`CreateCompletionAsStream`メソッドを利用し、画面描画までを行います。

[公式のサンプルはこちら](https://github.com/betalgo/openai/wiki/Chat-GPT)

まずは`CreateCompletionAsStream`メソッドの呼び出しです。

```csharp:StreamChatService.cs
public async IAsyncEnumerable<string> StreamChat(string prompt)
{
    var messages = new List<ChatMessage>()
    {
        ChatMessage.FromSystem("The following is a conversation with an AI assistant. The assistant is helpful, creative, clever, and very friendly."),
        ChatMessage.FromUser(prompt)
    };

    var completionResult = openAIService.ChatCompletion.CreateCompletionAsStream(
        new ChatCompletionCreateRequest()
        {
            Messages = messages,
            Model = Models.ChatGpt3_5Turbo
        });

    await foreach (var completion in completionResult)
    {
        if (completion.Successful)
        {
            yield return completion.Choices.FirstOrDefault()?.Message.Content ?? string.Empty;
        }
        else
        {
            if (completion.Error == null)
            {
                throw new Exception("Unknown Error");
            }
            Console.WriteLine($"{completion.Error.Code}: {completion.Error.Message}");
        }
    }
    Console.WriteLine("Complete");
}
```

注目すべきは、`StreamChat`メソッドの戻り値が `IAsyncEnumerable<string>` 型である点です。
これは非同期ストリームを表しており、レスポンスを受け取る度にその中身を呼び出し元に返すことが可能です。

参考文献： [非同期ストリーム - C# によるプログラミング入門](https://ufcpp.net/study/csharp/async/asyncstream/)

## Blazor Serverで非同期ストリームを受け取る:love_letter:

非同期ストリームから受けとった文字列を画面に表示させる場合、注意が必要です。

```csharp:StreamChat.razor

private async void GetChatCompletion()
{
    isChatting = true;
    responseMessage = string.Empty;
    
    var response = StreamChatService.StreamChat(chatMessage);
    await foreach(var blob in response)
    {
        responseMessage += blob;
        await InvokeAsync(StateHasChanged);
        
        // Allowing control to return to the renderer to update UI.
        await Task.Delay(1);
    }

    isChatting = false;
    await InvokeAsync(StateHasChanged);
}
```

コメントでも補足していますが、`await Task.Delay(1);`を実行し、制御をメインのイベントループに戻す必要があります。

[ComponentBase.cs](https://github.com/dotnet/aspnetcore/blob/5cfdaf9e0c1242ad8894feaa027653b0e085ebdd/src/Components/Components/src/ComponentBase.cs)がその実装箇所であり、新しいレンダリングをキューに入れ、実行を待機します。
どうやら処理を譲るだけで良さそうですので、`Task.Delay(1)`の長さは問題にはならなさそうですが、このあたりに詳しい方、是非ご教授いただきたいです。

## まとめ :feet:

`Betalgo.OpenAI`を用いて少しずつレスポンスを返す画面を実装しました。
`IAsyncEnumerable`の使い方やBlazorのレンダリングの仕組みについての理解が進んだのであれば幸いです。
最近ではChatGPTにFunction Calingの機能が追加されたりと、新しいことばかりですね。
また進展があれば記事にします:blush:
では:raised_hand:
