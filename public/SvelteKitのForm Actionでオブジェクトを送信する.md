---
title: SvelteKitのForm Actionでオブジェクトを送信する
tags:
  - TypeScript
  - Svelte
  - SvelteKit
private: false
updated_at: '2023-04-03T23:41:49+09:00'
id: 2ec6075d943ea400fb1e
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに 🐶

svelteKitの FormActionは、フォームの送信を制御するためのコンポーネントです。

ただし、通常のstring型のバインドでは、単なる文字列しか送信できません。
- [FormAction](https://kit.svelte.dev/docs/form-actions)

では、Form Actionを使ってオブジェクトを送信するにはどうすればいいのでしょうか。

## JSON文字列として送信する 🍉

どうやら現在のSvelteの実装ではフォームにオブジェクトを送信することはできないようです。
ということで、JSON文字列としてフォームに送信し、serverサイドでパースすることで実現します。

以下に実装例を示します。

まずは以下のようにフォームから送信するオブジェクトの型定義をします。
`ts
// 送信するオブジェクトの型定義
interface SampleObject {
  name: string;
  age: number;
}
`

次に、以下のようにフォームを実装します。

```html
<!-- src/routes/+paga.svelte -->
<script lang="ts">
	import type { ActionData } from './$types';
	import type { SampleObject } from './type';

	export let form: ActionData;

	let obj: SampleObject = {
		name: 'John Doe',
		age: 30
	};
</script>

<!-- 初期表示時はデータが存在しない -->
<div class="card p-4">
	name: {form?.name} <br/>
	age: {form?.age}
</div>

<form method="POST">
	<input type="hidden" name="object" value={JSON.stringify(obj)} />
	<button type="submit" class="btn btn-sm variant-filled-tertiary">Submit</button>
</form>

```

最後に、以下のように+page.server.tsを実装します。

```ts
// src/routes/+page.server.ts
import type { Actions } from './$types';
import type { SampleObject } from './type';

export const actions: Actions = {
	default: async ({ request }) => {
		const formData = await request.formData();
		const message = JSON.parse(formData.get('object')?.toString() as string) as SampleObject;

		return {
			name: message.name,
			age: message.age
		};
	}
};
```

上記の実装で、+page.server.tsに記述されたform actionを実行することができます。

🔽送信前
![Before request](https://raw.githubusercontent.com/TellMin/Qiita/main/before.png)

🔽送信後
![After request](https://raw.githubusercontent.com/TellMin/Qiita/main/after.png)

## 型を安全にパースする 📌

上記の実装では、JSON.parseでパースする際に、型を安全にパースすることができません。

そこで、以下のように型を安全にパースするための関数を実装します。

```ts
// src/routes/type.ts
export function isSampleObject(obj: unknown): obj is SampleObject {
  if (typeof obj !== 'object' || obj === null) {
    return false;
  }

  const maybeSampleObject = obj as SampleObject;

  return (
    typeof maybeSampleObject.name === 'string' &&
    typeof maybeSampleObject.age === 'number'
  );
}

export function parseSampleObject(jsonString: string): SampleObject | null {
  try {
    const parsedObj = JSON.parse(jsonString);

    if (isSampleObject(parsedObj)) {
      return parsedObj;
    } else {
      return null;
    }
  } catch (error) {
    return null;
  }
}

```

上記関数を+page.server.tsに実装します。

```ts
// src/routes/+page.server.ts
import type { Actions } from './$types';
import { parseSampleObject } from './type';

export const actions: Actions = {
	default: async ({ request }) => {
		const formData = await request.formData();
		const jsonString = formData.get('object')?.toString() ?? '';
		const message = parseSampleObject(jsonString);

		if (!message) {
			throw new Error('Invalid object format');
		}
		return {
			name: message.name,
			age: message.age
		};
	}
};
```

これで型を安全にしつつ、JSON文字列をパースすることができます。

## まとめ 🦀

Form Actionを使ってオブジェクトを送信することができました。
下記ユースケースでは有用かと思います。

- 配列や複雑なオブジェクトを送信する場合
- 画面が持つステートをサーバーに送信する場合

もちろん単純にAPIを用意し、fetchで送信する方法も取れます。

Form Actionの強みは`./$types`に定義された型を利用できることです。
その時々に応じて、適切な方法を選択すると良いと思います。

間違い、誤りがあればご指摘いただけると幸いです。

## 参考 🏺

- [FormAction](https://kit.svelte.dev/docs/form-actions)
