## 要件定義書（問い合わせ対応自動化AIエージェント）

### 1. プロジェクト概要

#### 1.1 目的
社内のローカルドキュメント（会社情報・サービス情報・顧客対応履歴）を参照し、ユーザーからの質問に回答するFAQチャットボット兼問い合わせ連携ツールを提供する。問い合わせモード時は、関連従業員へSlack通知を自動化し、社内対応を効率化する。

#### 1.2 スコープ
- StreamlitベースのWeb UIでの対話機能（チャットUI、サイドバー設定、フィードバック収集）
- ローカルファイル（PDF/Docx/TXT）およびCSVの読み込み・検索
- RAGによる質問応答（会話履歴考慮）
- 任意でAIエージェント（Tool連携・Web検索）による回答強化
- 問い合わせモード時のSlack通知（担当者選定＋メンション）

### 2. 機能要件

#### 2.1 ドキュメント管理（RAG）
- 対応フォーマット: PDF（PyMuPDF）、DOCX（docx2txt）、TXT（UTF-8）
- ファイル配置: `data/rag/{company|service|customer|all}` を自動走査
- 更新タイミング: アプリ起動時にロード。FAISSベクトル化をローカル保存（`.faiss_db`）
- チャンク分割: 文字数ベースで分割（500文字、オーバーラップ50）

#### 2.2 チャット機能
- 入力: テキストベース（`st.chat_input`）
- 処理:
  - 会話履歴を独立質問へ変換（history-aware retriever）
  - ベクトル検索（FAISS, k=5）
  - 必要に応じてTool経由の専門チェーン（会社/サービス/顧客/全体）を選択
  - LLMで回答生成（Markdown推奨）
- 出力:
  - 回答テキスト
  -（内部）ログ出力、フィードバック誘導
- フィードバック: はい/いいえ＋理由入力（ログ出力）

#### 2.3 基本設定
- LLMモデル: `gpt-4o-mini`
- 埋め込みモデル: `OpenAIEmbeddings`
- チャンクサイズ: 500、オーバーラップ: 50
- 検索結果数: 上位5件（RAG、エンsembleで可変）
- 会話トークン上限: 1000（超過時に古い履歴を削除）

#### 2.4 AIエージェント（任意）
- モード切替: サイドバーで「利用する/利用しない」
- 使用エージェント: ZERO_SHOT_REACT_DESCRIPTION
- 使用Tool:
  - 会社情報検索、サービス情報検索、顧客コミュニケーション検索、全体検索
  - Web検索（SerpAPI）

#### 2.5 問い合わせモード（Slack連携）
- モード切替: サイドバーで「ON/OFF」
- 担当者選定: 従業員情報・問い合わせ履歴（CSV）を取り込み、BM25＋FAISSのEnsembleで候補抽出、LLMでID厳選
- 通知内容: メンション先、問い合わせ情報（カテゴリ・日時）、選定理由、回答・対応案（3件×根拠）
- 通知先: Slack「動作検証用」チャンネル

### 3. 技術仕様

#### 3.1 使用ライブラリ（主要）
- streamlit, python-dotenv, logging
- langchain, langchain-openai, langchain-community
- openai, tiktoken
- faiss-cpu（ベクトルストア: FAISS）、（コメントでChromaへの切替可）
- PyMuPDF, docx2txt
- sudachipy（BM25前処理用の日本語分かち書き）
- serpapi（Web検索）
- slacktoolkit（LangChain SlackToolkit）

#### 3.2 アーキテクチャ（概略）
ユーザー入力
→ 質問の独立化（会話履歴考慮）
→ ベクトル検索（FAISS / Ensemble）
→ コンテキスト収集
→ LLM回答生成（gpt-4o-mini）
→ 回答表示・フィードバック
→（問い合わせモード時）Slack通知

#### 3.3 ディレクトリ構造（本プロジェクト）
```
dm_project_2/
├── main.py                # Streamlitエントリ
├── initialize.py          # 初期化（LLM/Agent/Logger/RAGチェーン）
├── components.py          # UI表示ロジック
├── utils.py               # RAG/Slack/補助関数群
├── constants.py           # 設定・定数
├── data/
│   └── rag/{all,company,service,customer}  # 参照ドキュメント
│
├── logs/application.log   # タイムローテーションログ
├── images/ai_icon.jpg
├── images/user_icon.jpg
└── requirements.txt ほか
```

### 4. 実装要件

#### 4.1 初期化処理
- `.env` 読み込み（なければ `st.secrets` からAPIキー類を設定）
- セッション状態の初期化（会話履歴、トークン合計、UI制御フラグ）
- ログ出力設定（TimedRotatingFileHandler, INFO）
- LLM生成（`gpt-4o-mini`）
- 専用RAGチェーン作成（会社/サービス/顧客/全体）
- Agent Executor作成（Tool登録、SerpAPI含む）

#### 4.2 質問応答処理
- retrieval: CharacterTextSplitter（chunk=500, overlap=50）＋ OpenAIEmbeddings
- vectorstore: FAISS（`.faiss_db`保存）
- retriever: k=5
- chain: history-aware retriever → stuff documents chain → retrieval chain
- エージェントON時はTool経由実行、OFF時はRAG直実行
- トークン上限超過時は古い履歴から順次削除

#### 4.3 問い合わせ処理（Slack）
- CSV読込（`data/slack/従業員情報.csv`, `data/slack/問い合わせ対応履歴.csv`）
- Unicode正規化・Windows互換文字調整
- BM25＋FAISSのEnsembleで候補抽出（k=5, 重み[0.5, 0.5]）
- LLMで従業員IDを厳選→SlackID抽出→メンション文字列生成
- LangChain SlackToolkit エージェントで投稿実行

#### 4.4 エラーハンドリング
- 初期化失敗、会話履歴表示失敗、主処理失敗、回答表示失敗時にログ＋UI表示
- APIキー未設定時は`.env`読み込み失敗時に`st.secrets`フォールバック
- ドキュメント未登録や検索不一致時は既定メッセージ表示（NO_DOC_MATCH）
- 入力文字数が上限超過時は受け付け拒否

### 5. 動作要件

#### 5.1 環境要件
- Python 3.11（本リポジトリの仮想環境基準）
- OpenAI APIキー、SerpAPIキー、Slack User Token
- メモリ: 1GB以上推奨
- ストレージ: 1GB以上（ベクトルDB・ログ）

#### 5.2 制限事項
- 同時実行は1セッションのみを想定
- 参照ドキュメントは実運用で合計100MB程度までを目安
- 1回の質問は1000トークン相当以内（アプリ内部の上限連動）
- 標準回答は数十秒以内を目安（エージェントON・Web検索時は延伸）

### 6. セキュリティ・運用
- APIキーは`.env`または`st.secrets`で管理し、コード直書き禁止
- ログは`logs/application.log`に日次ローテーション、個人情報を出力しない
- PDFやCSVはUTF-8（BOM付CSVは`utf-8-sig`）を前提

### 7. 受け入れ条件（抜粋）
- RAGで`data/rag`配下の資料が検索・回答に反映される
- サイドバーでエージェント切替と問い合わせモード切替が機能する
- 問い合わせモードON時、指定チャンネルへメンション付き投稿が行われる
- 主要エラー時にUIで適切に通知され、ログへ記録される


