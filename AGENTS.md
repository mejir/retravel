# Retravel

設計レビューは [統合設計案](retravel-design.md) から始める。現状は travel-itinerary の確認済みスナップショットに設計書を追加した段階。統合機能は実装前。

- 土台のソースは [travel-itinerary e87361b](https://github.com/mejir/travel-itinerary/tree/e87361b0d95a8e442498380fb6bd6421ad3603f0)。精算の参照元は [warikan 0a2a692](https://github.com/mejir/warikan/tree/0a2a692fd86d0c75a53bd4a48b9ff96d82fd7d55)。
- 初回は設計書の段階0を確認し、ユーザーが指定した段階から実装する。実装は作業ブランチで進める。
- 作業環境では Node.js 22.12 以上を使い、`npm ci` で依存関係を入れる。コードの確認は `npm run lint` と `npm run build`。Firebase の動作確認にはエミュレーターまたは分離した検証環境を使う。
- 既存 Firebase Rules は旧アプリの調査用。既存の両リポジトリと本番 Firebase の設定・データへの変更、外部サービスの有料化、デプロイはユーザーの指示があるまで行わない。本番の認証情報や旅行データをコミットしない。
