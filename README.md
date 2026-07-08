#ユースケース図風の図
    graph LR
    %% アクターの定義
    subgraph アクター
        User((就活生))
    end

    %% アプリケーション境界とユースケース
    subgraph 就活特化予定管理アプリ
        UC1(アプリを起動する)
        UC2(1週間の予定を確認する)
        UC3(締切間近のリマインドを確認する)
        
        UC4(予定を登録・編集する)
        UC5(予定を削除・完了する)
        
        UC6(カレンダー表示を切り替える)
        
        %% 内部的なシステムユースケース
        UC_Save((ローカルにデータを保存する))
    end

    %% アクターとユースケースの接続
    User --> UC1
    User --> UC4
    User --> UC5
    User --> UC6

    %% 関係性（include / extend）の定義
    
    %% アプリ起動時に必ず1週間予定とリマインドが表示される (include)
    UC1 -.->|<< include >>| UC2
    UC1 -.->|<< include >>| UC3

    %% 登録・編集・削除時には必ずローカル保存が走る (include)
    UC4 -.->|<< include >>| UC_Save
    UC5 -.->|<< include >>| UC_Save

    %% 予定登録時に、条件（締切が近い）を満たした場合のみリマインドに反映される (extend)
    UC3 -.->|<< extend >>| UC4

    %% スタイル定義（見やすさのため）
    style User fill:#f9f,stroke:#333,stroke-width:2px
    style UC_Save fill:#e1f5fe,stroke:#0288d1,stroke-width:1px,stroke-dasharray: 5 5

#クラス図
    classDiagram
    direction TB

    %% クラスの定義
    class AppController {
        -currentView: String
        +initApp() Void
        +refreshDisplay() Void
        +switchView(viewName: String) Void
    }

    class TaskManager {
        -tasks: List~Task~
        +addTask(task: Task) Boolean
        +updateTask(taskId: String, updatedTask: Task) Boolean
        +deleteTask(taskId: String) Boolean
        +getWeeklyTasks() List~Task~
        +getUrgentTasks(daysBefore: Number) List~Task~
    }

    class Task {
        -id: String
        -companyName: String
        -deadline: Date
        -content: String
        -isCompleted: Boolean
        +toggleComplete() Void
        +isUrgent(daysBefore: Number) Boolean
    }

    class StorageManager {
        -storageKey: String
        +save(tasks: List~Task~) Boolean
        +load() List~Task~
    }

    %% クラス間の関連・多重度
    AppController "1" --> "1" TaskManager : 制御・利用
    AppController "1" --> "1" StorageManager : 起動時/変更時のデータ命令
    
    %% TaskManagerとTaskはコンポジション関係（Managerが消えればTaskリストも消える、ライフサイクルが共にある）
    TaskManager "1" *-- "0..*" Task : 管理する >
    
    %% StorageManagerはTaskクラスのデータを扱う
    StorageManager ..> Task : 依存（シリアライズ/デシリアライズ）

#シーケンス図
    sequenceDiagram
    autonumber
    actor User as 就活生 (Actor)
    participant UI as アプリ画面 (UI)
    participant Ctrl as AppController (Controller)
    participant Model as TaskManager / Storage (Model)

    User ->> UI: アプリを起動する
    activate UI
    UI ->> Ctrl: initApp()
    activate Ctrl
    
    Ctrl ->> Model: load() [データ読み込み]
    activate Model
    Model -->> Ctrl: タスクリストを返却
    deactivate Model

    alt タスクデータが存在する場合
        Ctrl ->> Model: getWeeklyTasks()
        activate Model
        Model -->> Ctrl: 1週間以内のタスク一覧
        deactivate Model

        Ctrl ->> Model: getUrgentTasks(3) [直近3日以内]
        activate Model
        Model -->> Ctrl: 緊急タスク一覧
        deactivate Model

        Ctrl ->> UI: 1週間予定 ＆ リマインドエリアを描画
    else データが空（初回起動など）の場合
        Ctrl ->> UI: 「予定がありません。登録してください」を表示
    end

    Ctrl -->> UI: 描画完了
    deactivate Ctrl
    UI -->> User: 1週間の予定とリマインドが表示される
    deactivate UI

#状態遷移図
    stateDiagram-v2
    [*] --> 新規作成状態 : ユーザーが予定を入力

    state 新規作成状態 {
        [*] --> 未完了_締切前 : ローカルストレージに保存
    }

    未完了_締切前 --> 未完了_締切前 : 予定を編集 [トリガー: 変更保存]
    未完了_締切前 --> 完了_提出済 : 提出を完了する [トリガー: 完了チェックON]

    %% 時間経過による自動遷移
    未完了_締切前 --> 未完了_締切超過 : 締切日時を過ぎる [トリガー: アプリ起動時の時間判定]

    未完了_締切超過 --> 完了_提出済 : 遅れて提出/完了処理 [トリガー: 完了チェックON]
    未完了_締切超過 --> 未完了_締切前 : 締切日数を延長する [トリガー: 変更保存]

    %% 完了から未完了への戻り
    完了_提出済 --> 未完了_締切前 : チェックを誤って外す [トリガー: 完了チェックOFF ＆ 締切前]
    完了_提出済 --> 未完了_締切超過 : チェックを誤って外す [トリガー: 完了チェックOFF ＆ 締切超過]

    %% 終了状態への遷移（削除）
    未完了_締切前 --> [*] : 予定を削除する [トリガー: 削除ボタン押下]
    未完了_締切超過 --> [*] : 予定を削除する [トリガー: 削除ボタン押下]
    完了_提出済 --> [*] : 予定を削除する（または古いデータを一括削除） [トリガー: 削除ボタン押下]
