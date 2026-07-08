---
title: "PlaywrightのE2EテストをPOM+Fixtureで設計した話"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["playwright", "e2e", "typescript", "qa", "testautomation"]
published: true
---

## はじめに

QAエンジニアとして働きながら、業務外でPlaywrightを使ったE2Eテストの自動化を学習しています。
今回、学習用に作ったシンプルなTodoアプリに対してE2Eテストを実装し、[GitHubに公開](https://github.com/tatsuo-0/todo-app-playwright)しました。

この記事では、テストコードそのものよりも「どう設計したか」「なぜそう設計したか」に焦点を当てて書きます。
題材が小さいアプリなので大げさな設計には見えないかもしれませんが、実務のE2Eテストでも同じ考え方がそのまま使えると考えています。

対象読者:
- Playwrightでこれからテストを書き始める人
- Page Object ModelやFixtureを「なんとなく」導入しているが、狙いを言語化したい人

## 作ったもの

- テスト対象: シンプルなTodoアプリ（追加・完了・削除ができるだけのSPA）
- テストケース: Todo追加／完了／削除の3機能
- 技術構成: Playwright + TypeScript、GitHub Actionsで3ブラウザ（Chromium/Firefox/WebKit）を自動実行

## 設計のポイント1: Page Object Modelでロケータを一箇所に集約する

テストコードの中に `page.getByRole(...)` を直接書いていくと、UIが変わるたびに全テストファイルを修正することになります。
それを避けるため、画面操作を `TodoPage` クラスに閉じ込めました。

```ts
export class TodoPage {
  readonly page: Page;
  readonly input: Locator;
  readonly addButton: Locator;
  readonly listItems: Locator;

  constructor(page: Page) {
    this.page = page;
    this.input = this.page.getByRole('textbox', { name: 'やることを入力' });
    this.addButton = this.page.getByRole('button', { name: '追加' });
    this.listItems = this.page.getByRole('listitem');
  }

  async open() {
    await this.page.goto('/');
  }

  async add(text: string) {
    await this.input.fill(text);
    await this.addButton.click();
  }

  getItem(text: string) {
    return this.listItems.filter({ hasText: text });
  }

  async toggle(text: string) {
    const item = this.getItem(text);
    await item.getByRole('checkbox').check();
  }

  async delete(text: string) {
    const item = this.getItem(text);
    await item.getByRole('button').filter({ hasText: /^$/ }).click();
  }
}
```

ポイントは、ロケータの取得方法として `getByRole` を使い、アクセシビリティツリーに基づいた操作にしていることです。
`data-testid` を振る方法も検討しましたが、あえてrole基準に統一しました。理由は2つあります。

1. `data-testid` はテスト専用の目印なので、実際のユーザーが認識できる情報（ボタンの役割・ラベル文言）とテストが一致しているとは限りません。role基準にすると「ユーザーが見ている画面」に沿ってテストが書けます
2. 副次的に、ボタンや入力欄に適切な `role` やアクセシブルネームが付いているかをテストが検証してくれるため、アクセシビリティのチェックを兼ねられます

テスト側は `TodoPage` のメソッドを呼ぶだけになり、「何をテストしているか」がテストコード上でそのまま読めます。

```ts
test('todoを追加できること', async ({ todoPage }) => {
  await test.step('Todo追加', async () => {
    await todoPage.add('参考書を開く');
  });
  await test.step('Todo追加すること', async () => {
    await expect(todoPage.getItem('参考書を開く')).toBeVisible();
  });
});
```

`test.step` でテストの意図をステップ単位に分けているのも、レポートを見たときに失敗箇所をすぐ特定できるようにするためです。

## 設計のポイント2: Fixtureでセットアップを共通化する

各テストの冒頭で「ページを開く」処理を毎回書くと重複が増えるので、Playwrightのfixture機能で共通化しました。

```ts
export const test = base.extend<{ todoPage: TodoPage }>({
  todoPage: async ({ page }, use) => {
    const todoPage = new TodoPage(page);
    await todoPage.open();
    await use(todoPage);
  },
});
```

これで各テストは `({ todoPage }) => { ... }` と書くだけで、初期化済みの `TodoPage` を受け取れます。
「テストごとの前処理」をテスト本文から追い出せるので、テストケースが本来のアサーションに集中できるのがメリットです。

## 寄り道: Worker Fixtureを試して、あえて使わなかった話

学習の一環として、worker単位でページを使い回す `worker fixture` も実装しました。

```ts
export const test = base.extend<{}, { sharedTodoPage: TodoPage }>({
  sharedTodoPage: [
    async ({ browser }, use) => {
      const page = await browser.newPage();
      const todoPage = new TodoPage(page);
      await todoPage.open();
      await todoPage.add('初期データ1');
      await todoPage.add('初期データ2');
      await use(todoPage);
      await page.close();
    },
    { scope: 'worker' },
  ],
});
```

通常のfixtureは「テストごと」に生成されるのに対し、`{ scope: 'worker' }` を指定するとworker（並列実行の単位）ごとに1回だけ生成され、複数のテストで使い回されます。
実行速度は上がりますが、あるテストが状態を変更すると別のテストに影響してしまうため、今回のような「追加・完了・削除」を検証するテストとは相性が悪いと判断し、実際のテストスイートでは採用しませんでした。

このリポジトリには使っていないコードが残っていますが、「速度のためにテスト間の独立性を犠牲にするかどうか」を自分の手で試して判断できたのは収穫でした。

## つまずいた点

`playwright.config.ts` の `webServer` 設定です。

```ts
webServer: {
  command: 'npm run dev',
  url: 'http://localhost:5173',
  reuseExistingServer: !process.env.CI,
},
```

ローカルでは既に立ち上がっている開発サーバーを再利用できますが、GitHub Actions上ではサーバーが存在しないため、CI環境では毎回 `npm run dev` を実行してサーバーが立ち上がるのを待ってからテストを開始する必要があります。

最初は `webServer` の設定が抜けており、ローカルでは（手元でdevサーバーを立てたまま実行していたため）問題なく通っていたテストが、CI上では `page.goto: net::ERR_CONNECTION_REFUSED` のようなエラーで軒並み失敗しました。原因はシンプルで、CIにはアプリを起動するプロセスが存在しないため、`baseURL` に接続できていなかっただけです。`webServer` にコマンドとURLを指定し、CIでも自動でdevサーバーを起動・待機してからテストを開始するようにして解決しました。
ローカルで通ったから大丈夫、とはいかないことを実感した箇所です。

## まとめ

- Page Object Modelでロケータとテストコードの責務を分離した
- Fixtureで「テストごとの前処理」を共通化した
- Worker Fixtureは試した上で、テストの独立性を優先して不採用にした
- CI特有の環境差分（`webServer` の挙動）はローカルだけでは気づけない

次はGitHub Actions側の設定（3ブラウザ並列実行・レポート保存）について書く予定です。
