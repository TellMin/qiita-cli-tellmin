---
title: SvelteKit初心者がForm actionsで悩んでたら、HTMLの基本を思い出した話
tags:
  - HTML
  - TypeScript
  - Svelte
  - SvelteKit
private: false
updated_at: '2023-03-21T23:55:05+09:00'
id: 287b27c537f35a20e669
organization_url_name: null
slide: false
ignorePublish: false
---
## TL;DR :chocolate_bar: 

:warning: \<form\>で`submit`イベントを発火させるときは\<button\>に`type="submit"`が指定されていることを確認しよう:exclamation: 

> あるいは`type`になにも指定しなければ自動で`submit`になります。なにも指定しなければ。

### この記事は何？ :dango:

`GitHub Copilot` :airplane: に頼りすぎた愚かな開発者（筆者）がHTMLの基本を忘れたことを戒めるための記事です。

### 起きたこと :book: 

それは`SvelteKit`の[Form Actions](https://kit.svelte.dev/docs/form-actions)学習中のことでした。
筆者は`GitHub Copilot` :airplane: を使いながら、以下のコードを作成しました。

```svelte +page.svelte //.html
<script lang="ts">
	import type { ActionData } from './$types';
	export let form: ActionData;
</script>

<h1>{form}</h1>

<form method="POST">
	<button type="button">
        Click
    </button>
</form>
```

```typescript +page.server.ts
import { error } from '@sveltejs/kit';
import type { Actions } from './$types';

export const actions: Actions = {
	default: async () => 'Hello World!'
};
```
(`.svelte`もシンタックスハイライト対応しないかな...)

ここで詳しい話は割愛いたしますが、`+page.svelte`で`form`が`submit`された場合、`+page.server.ts`で定義した`Form  Actions`が実行され、画面に"Hello World!"と出力されるのが期待される動作です。

しかし実際には`{form}`は`null`のまま動きません。

### 解決方法 :coffee: 

```html
    <-- 下記コードを -->
	<button type="button">
        Click
    </button>
    
    <-- こうする -->
	<button>
        Click
    </button>
```

### 特に必要のない解説 :bee: 

あまり説明することもないのですが、`<button type="button" />`と、ボタンの`type`に`button`が指定されている場合、`submit`する能力を失ってただのボタンになっています。
そのためイベントが発火せず、`Form`が送信されなかったということです。

> ボタンの`type`が指定されていない場合の初期値は`submit`です。[form 内の button 要素を押すと submit される理由](https://zenn.dev/phi/articles/form-submit-button-type-default)
> `type`が`button`の場合でも、イベントハンドラーを登録することで発火させることが可能

## まとめ :beach_umbrella: 

`GitHub Copilot` :airplane: は非常に頼れる開発ツールですが、盲目的になってはいけないと再確認しました。
少なくとも、出力された結果に対してしっかり理解する姿勢が必要です。

ちなみに原因特定は`ChatGPT` にお願いしました。
便利な世の中になったものです。いや、ほんと。
