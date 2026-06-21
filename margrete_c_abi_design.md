# marmkmt C ABI 仕様書

## 1. 適用範囲

本書は、Margrete Plugin SDKのC++ ABIを、他言語から利用可能な安定C ABIへ変換するstatic library `marmkmt` の規範仕様である。

規範語「必須」「禁止」「許可」はABI実装および言語wrapperが従う要件を示す。

対象一次資料は次の通りである。

- Margrete Plugin SDK `master`、`MP_SDK_VERSION == 2`、commit `b8d0c87125e090880a75f6f0ff57a773510299a8`
- Margrete Plugin Sample `master`、commit `cfe097756d2ba777354977e896b261e16862f706`

Context7にはMargrete Plugin SDKが登録されていないため、上記の公式ヘッダと公式サンプルを正本とする。SDK更新時は最新版の公式ヘッダへ追従し、旧SDKとの同時互換は提供しない。

## 2. 配布物、外部wrapper、最終配置物

### 2.1 marmkmt本体の配布物

marmkmt本体は、各言語wrapper開発者が最終的なMargrete plugin DLLを構築するためのネイティブライブラリである。本体のバイナリ成果物は、x64 Windows向けstatic library `marmkmt.lib` 1つとする。

公式配布物には次を含める。

- 正規Cヘッダ
- x64 Windows向けstatic library `marmkmt.lib`
- ライセンスなど、配布に必要なメタデータ

marmkmtは最終的なMargrete plugin DLLへ静的リンクする。marmkmt本体のDLL版はビルドまたは配布しない。bridge DLL、別impl DLL、`.cabi.ini`、共通runner、動的な言語DLL探索は使用しない。

公式配布物に各言語wrapper、plugin template、サンプルpluginの実装を含めない。

### 2.2 外部の言語wrapper

Rustの `marmkmt-rs` など、各言語のwrapperはC ABIをその言語向けの安全で自然なAPIへ変換する、marmkmt本体とは独立した外部成果物である。wrapperのソース、バイナリ、template、サンプルpluginはそれぞれのプロジェクトが配布する。それらの互換性保証は各プロジェクトで定め、marmkmt本体のABI保証には含めない。

通常のC# class libraryやPython scriptはC ABI実装ではない。各wrapperはNativeAOT、native extension、埋め込みruntime等、その言語で成立する方式を選ばなければならない。

### 2.3 最終配置物

Margreteがロードする最終配置物は、wrapper開発者またはplugin開発者が構築する単一のplugin DLLとする。このDLLはmarmkmt本体の配布物ではない。このDLLは次のコードとシンボルを統合する。

- marmkmt static libraryから静的リンクしたコード
- 言語wrapper
- plugin実装
- `MargretePluginGetInfo`
- `MargretePluginCommandCreate`
- `mg_plugin_init`

1 plugin DLLは1種類のcommandを提供する。同じ種類のcommand instanceが複数作られることは許可する。複数種類のcommandを提供する場合はDLLを分ける。

## 3. 対象環境とバージョン

- OS: Windows
- architecture: x64のみ
- Margrete SDK: ビルド時点の公式最新版のみ
- 初期基準SDK: `MP_SDK_VERSION == 2`
- C ABI開発版: `0.x`。破壊的変更を許可する
- 安定版: ABI 1。凍結後は本書の互換性規則に従う

ビルド成果物には対象 `MP_SDK_VERSION` とSDK commitを記録する。SDKの整数バージョンとmarmkmt C ABI versionは別に管理する。

## 4. 境界と呼び出し規約

Margrete側exportは公式SDKどおり `extern "C"` と `__stdcall` を使用する。

```cpp
void __stdcall MargretePluginGetInfo(MP_PLUGININFO* info);
MpBoolean __stdcall MargretePluginCommandCreate(IMargretePluginCommand** out_command);
```

C ABIの関数とcallbackはx64 Windowsの通常のC呼び出し規約を使用する。ヘッダでは次のマクロで表す。

```c
#if defined(_WIN32)
#define MG_CALL __cdecl
#else
#define MG_CALL
#endif
```

C ABI境界にC++型、C++例外、Rust panic、RTTI、template、参照、標準ライブラリ型を露出してはならない。

## 5. 初期化とplugin callback

言語wrapperは最終plugin DLL内に次のシンボルを提供する。

```c
MgResult MG_CALL mg_plugin_init(
    uint32_t requested_abi_version,
    const MgHostApi* host_api,
    MgPluginApi* out_plugin_api
);
```

marmkmtは最初のplugin情報要求またはcommand生成要求までに、この関数をスレッドセーフに一度だけ呼ぶ。`DllMain` から呼んではならない。

`out_plugin_api` は呼び出し前にゼロ初期化し、marmkmtが `struct_size` と要求ABI versionを設定する。plugin側は受信した `struct_size` を超えて書き込んではならない。初期化失敗時はダイアログを表示し、`MargretePluginCommandCreate` は `MP_FALSE` を返す。

```c
typedef struct MgPluginInfoBuffers {
    uint32_t struct_size;
    char* name;
    uint32_t name_capacity;
    uint32_t* name_required;
    char* description;
    uint32_t description_capacity;
    uint32_t* description_required;
    char* developer;
    uint32_t developer_capacity;
    uint32_t* developer_required;
} MgPluginInfoBuffers;

typedef struct MgPluginApi {
    uint32_t struct_size;
    uint32_t abi_version;
    void* plugin_data;

    MgResult (MG_CALL *get_plugin_info)(
        void* plugin_data,
        MgPluginInfoBuffers* buffers);

    MgResult (MG_CALL *create_command)(
        void* plugin_data,
        void** out_command);

    void (MG_CALL *destroy_command)(
        void* plugin_data,
        void* command);

    MgResult (MG_CALL *get_command_name)(
        void* plugin_data,
        void* command,
        char* buffer,
        uint32_t capacity,
        uint32_t* out_required);

    MgResult (MG_CALL *invoke)(
        void* plugin_data,
        void* command,
        MgContext* context);
} MgPluginApi;
```

全callbackは必須である。`destroy_command` はno-failとし、例外やpanicを外へ出してはならない。

公式サンプルに従い、plugin名、説明、開発者名は `MargretePluginGetInfo` へ、command名は `getCommandName` へ転送する。C ABI側のUTF-8文字列を正本とし、marmkmtがUTF-16へ変換する。ABI 1はlocale別文字列を提供しない。

## 6. API tableの互換性

`MgHostApi` と `MgPluginApi` の先頭は必ず次の順序とする。

```c
uint32_t struct_size;
uint32_t abi_version;
```

- 呼び出し側は自分が確保したサイズを `struct_size` に設定する。
- 書き込み側は `struct_size` を超えて書き込んではならない。
- 受信側は未知の末尾フィールドを無視する。
- 必須fieldは、受信したサイズ内にあり非NULLでなければならない。
- ABI 1凍結後、fieldの削除、並べ替え、意味変更は禁止する。
- ABI 1凍結後の追加は末尾の任意fieldだけを許可する。
- 定数値を別の意味へ再利用してはならない。

## 7. 基本型とエラー

```c
typedef int32_t MgResult;
typedef int32_t MgBool;
typedef int32_t MgInt;
typedef int32_t MgEventKind;

#define MG_FALSE 0
#define MG_TRUE 1

#define MG_OK 0
#define MG_ERR_NULL_POINTER -1
#define MG_ERR_INVALID_ARGUMENT -2
#define MG_ERR_INVALID_HANDLE -3
#define MG_ERR_WRONG_THREAD -4
#define MG_ERR_BUFFER_TOO_SMALL -5
#define MG_ERR_UNSUPPORTED_ABI -6
#define MG_ERR_PLUGIN_INIT_FAILED -7
#define MG_ERR_SDK_FAILURE -8
#define MG_ERR_PANIC -9
#define MG_ERR_INTERNAL -10
#define MG_ERR_COMMAND_POISONED -11
#define MG_ERR_NOT_FOUND -12
```

公開ABIではCの名前付き `enum`、`bool`、`long`、`size_t` を使用しない。`MgBool` の入力は0をfalse、非0をtrueとし、出力は0または1へ正規化する。

全関数は次に従う。

- 必須pointerがNULLなら `MG_ERR_NULL_POINTER`。
- object out引数は処理前にNULLへ初期化する。
- scalar out引数は処理前に0へ初期化する。
- 失敗時に部分成功したout値を返さない。
- 負のindex、範囲外index、未知のkind、非有限の浮動小数点値を拒否する。
- SDKのvoid methodはC++例外がなければ `MG_OK` とする。SDK内部の論理失敗は検出できない。
- `findEvent*`、`getParent`、`getBaseNote` 等で対象が存在しないことは `MG_ERR_NOT_FOUND` とし、障害として扱わない。

## 8. 文字列

C ABI文字列は検証済みUTF-8とする。

- lengthとcapacityはNULを含むbyte数である。
- 空文字列の必要長は1である。
- 埋め込みNULは禁止する。
- `buffer == NULL` かつcapacity 0をサイズ照会として許可する。
- size照会は `MG_OK` と必要長を返す。
- capacity不足は `MG_ERR_BUFFER_TOO_SMALL` と必要長を返す。
- UTF-8またはUTF-16 code pointの途中で切断してはならない。

Margrete SDKの `wchar_t` buffer長はNULを含む `wchar_t` 要素数、WindowsではUTF-16 code unit数として扱う。

## 9. handle、所有権、寿命

```c
typedef struct MgContext MgContext;
typedef struct MgDocument MgDocument;
typedef struct MgChart MgChart;
typedef struct MgNote MgNote;
typedef struct MgEvent MgEvent;
typedef struct MgUndoBuffer MgUndoBuffer;
typedef void* MgNativeWindowHandle;
```

公開handleを生のSDK interface pointerとしてはならない。marmkmt内部handleは少なくとも種別、SDK pointer、所有区分、生存状態、作成threadを保持する。

- `MgContext` は `invoke` 中だけ有効な借用handleであり、releaseしてはならない。
- Contextから取得した他のhandleは所有handleである。
- 所有handleは `invoke` 終了前にすべてreleaseしなければならない。
- 入力handleを受け取る操作は、明記がない限り所有権を消費しない。
- append/delete後も呼び出し側の所有handleはrelease責任を持つ。
- NULL、型違い、解放済みhandleは `MG_ERR_INVALID_HANDLE`。
- handleをplugin永続状態へ保存してはならない。

SDKの `queryInterface` は公開しない。ABI 1が対象とする全interfaceを型付きhandle、`chart_create_event`、`event_get_kind` で表現する。これにより現行SDKの全機能を提供しつつGUIDとC++ interfaceをC ABIから隠蔽する。

## 10. threadと再入

- SDKに触れる全操作とreleaseは、対応する `invoke` threadで行う。
- handleを別threadへ渡してはならない。
- 別threadへ渡せるのは値copyしたデータだけである。
- worker threadを使う場合は `invoke` 中にjoinする。
- 非同期結果を `invoke` 復帰後にMargreteへ反映する機能はABI 1の対象外である。
- 同じcommand instanceへの同時invokeと再入を禁止する。
- Rust wrapperのhandleは `!Send` かつ `!Sync` とする。
- GC finalizerからSDKのreleaseを呼んではならない。

## 11. 例外、panic、障害通知

C++例外とunwind可能なpanicは必ずABI境界で捕捉する。捕捉できた場合、Margrete全体を終了させる必要はない。

- callbackが例外またはpanicを発生させたcommand instanceはpoisoned状態にする。
- 発生時は短いエラー内容をダイアログで表示する。
- poisoned instanceへの以後のinvokeはcallbackを呼ばず、ダイアログを表示して `MG_ERR_COMMAND_POISONED` 相当として失敗させる。
- commandを破棄して新しいinstanceが生成された場合、poisoned状態は引き継がない。
- wrapperはRustの `panic=abort` 等、unwind不能設定を公式構成に使用してはならない。
- `get_plugin_info`、`create_command`、`get_command_name`、`invoke` callbackが通常の失敗結果を返した場合も、marmkmtはダイアログで通知する。
- Host API内部の `MG_ERR_NOT_FOUND` は正常な検索不成立であり、自動的にダイアログを表示しない。pluginが最終的に処理失敗と判断した場合は `report_error` を呼ぶか、`invoke` から失敗結果を返す。

アクセス違反、stack破壊、heap破壊、abort、プロセス終了は安全に回復できない。この種の障害ではMargrete全体が終了する可能性があり、marmkmtは隔離を保証しない。第三者pluginは信頼されたin-process native codeとして扱う。

ABI 1はファイルログを出力しない。通常動作、失敗、例外、panicのいずれもファイルへ記録せず、失敗通知はダイアログへ一本化する。

`invoke` 中は `context_get_main_window_handle` で得たMargrete windowをdialog ownerにする。それ以前の初期化失敗ではownerなしのdialogを許可する。

## 12. 公開データ構造

```c
typedef struct MgNoteInfo {
    MgInt type;
    MgInt long_attr;
    MgInt direction;
    MgInt ex_attr;
    MgInt variation_id;
    MgInt x;
    MgInt width;
    MgInt height;
    MgInt tick;
    MgInt timeline_id;
    MgInt option_value;
} MgNoteInfo;

typedef struct MgEventTimelineSpeedInfo {
    MgInt timeline_id;
    MgInt tick;
    double speed;
} MgEventTimelineSpeedInfo;

typedef struct MgEventNoteSpeedModifierInfo {
    MgInt tick;
    double speed;
} MgEventNoteSpeedModifierInfo;

typedef struct MgEventBpmInfo {
    MgInt tick;
    double bpm;
} MgEventBpmInfo;

typedef struct MgEventBeatChangeInfo {
    MgInt bar;
    MgInt beats_per_bar;
    MgInt beat_unit;
} MgEventBeatChangeInfo;
```

公開構造体へ無条件の `#pragma pack(1)` を適用してはならない。正規ヘッダと全公式wrapperはx64における `sizeof`、alignment、全fieldのoffsetをテストする。

## 13. 定数

定数は固定幅整数のmacroとして定義し、公式SDKの値と一致させる。

```c
#define MG_NOTE_TYPE_UNKNOWN 0
#define MG_NOTE_TYPE_TAP 1
#define MG_NOTE_TYPE_EXTAP 2
#define MG_NOTE_TYPE_FLICK 3
#define MG_NOTE_TYPE_DAMAGE 4
#define MG_NOTE_TYPE_HOLD 5
#define MG_NOTE_TYPE_SLIDE 6
#define MG_NOTE_TYPE_AIR 7
#define MG_NOTE_TYPE_AIRHOLD 8
#define MG_NOTE_TYPE_AIRSLIDE 9
#define MG_NOTE_TYPE_AIRCRUSH 10
#define MG_NOTE_TYPE_CLICK 11

#define MG_NOTE_LONG_ATTR_NONE 0
#define MG_NOTE_LONG_ATTR_BEGIN 1
#define MG_NOTE_LONG_ATTR_STEP 2
#define MG_NOTE_LONG_ATTR_CONTROL 3
#define MG_NOTE_LONG_ATTR_CURVE_CONTROL 4
#define MG_NOTE_LONG_ATTR_END 5
#define MG_NOTE_LONG_ATTR_END_NOACT 6

#define MG_NOTE_DIRECTION_NONE 0
#define MG_NOTE_DIRECTION_AUTO 1
#define MG_NOTE_DIRECTION_UP 2
#define MG_NOTE_DIRECTION_DOWN 3
#define MG_NOTE_DIRECTION_CENTER 4
#define MG_NOTE_DIRECTION_LEFT 5
#define MG_NOTE_DIRECTION_RIGHT 6
#define MG_NOTE_DIRECTION_UP_LEFT 7
#define MG_NOTE_DIRECTION_UP_RIGHT 8
#define MG_NOTE_DIRECTION_DOWN_LEFT 9
#define MG_NOTE_DIRECTION_DOWN_RIGHT 10
#define MG_NOTE_DIRECTION_ROTATE_LEFT 11
#define MG_NOTE_DIRECTION_ROTATE_RIGHT 12
#define MG_NOTE_DIRECTION_IN_OUT 13
#define MG_NOTE_DIRECTION_OUT_IN 14

#define MG_NOTE_EX_ATTR_NONE 0
#define MG_NOTE_EX_ATTR_INVERT 1
#define MG_NOTE_EX_ATTR_HAS_NOTE 2
#define MG_NOTE_EX_ATTR_EXJDG 3

#define MG_OPTION_AIRCRUSH_TRACE_LIKE 0
#define MG_OPTION_AIRCRUSH_HEAD_ONLY INT32_MAX

#define MG_EVENT_KIND_TIMELINE_SPEED 1
#define MG_EVENT_KIND_NOTE_SPEED_MODIFIER 2
#define MG_EVENT_KIND_BPM 3
#define MG_EVENT_KIND_BEAT_CHANGE 4
```

SDKに定義される `MP_NOTETYPE_LAST == 13` は有効なnote typeではなく番兵値であるため公開APIの入力として受理しない。

## 14. 完全Host API

ABI 1は基準SDKの全操作を提供する。次が正規 `MgHostApi` のfield順序である。説明用に同じ構造体を複数箇所へ分割定義してはならない。

```c
typedef struct MgHostApi {
    uint32_t struct_size;
    uint32_t abi_version;

    MgResult (MG_CALL *object_add_ref)(void* object);
    MgResult (MG_CALL *object_release)(void* object);

    MgResult (MG_CALL *report_error)(
        MgContext* context,
        const char* utf8_message,
        uint32_t message_length);

    MgResult (MG_CALL *context_get_document)(MgContext*, MgDocument**);
    MgResult (MG_CALL *context_get_main_window_handle)(MgContext*, MgNativeWindowHandle*);
    MgResult (MG_CALL *context_get_current_tick)(MgContext*, MgInt*);
    MgResult (MG_CALL *context_update)(MgContext*);

    MgResult (MG_CALL *document_get_chart)(MgDocument*, MgChart**);
    MgResult (MG_CALL *document_get_undo_buffer)(MgDocument*, MgUndoBuffer**);

    MgResult (MG_CALL *undo_begin_recording)(MgUndoBuffer*);
    MgResult (MG_CALL *undo_commit_recording)(MgUndoBuffer*);
    MgResult (MG_CALL *undo_discard_recording)(MgUndoBuffer*);
    MgResult (MG_CALL *undo_undo)(MgUndoBuffer*);
    MgResult (MG_CALL *undo_redo)(MgUndoBuffer*);
    MgResult (MG_CALL *undo_can_undo)(MgUndoBuffer*, MgBool*);
    MgResult (MG_CALL *undo_can_redo)(MgUndoBuffer*, MgBool*);
    MgResult (MG_CALL *undo_is_recording)(MgUndoBuffer*, MgBool*);

    MgResult (MG_CALL *chart_create_note)(MgChart*, MgNote**);
    MgResult (MG_CALL *chart_get_notes_count)(MgChart*, MgInt*);
    MgResult (MG_CALL *chart_get_note)(MgChart*, MgInt index, MgNote**);
    MgResult (MG_CALL *chart_append_note)(MgChart*, MgNote*);
    MgResult (MG_CALL *chart_delete_note)(MgChart*, MgNote*);
    MgResult (MG_CALL *chart_offset_notes)(MgChart*, MgInt tick);

    MgResult (MG_CALL *note_get_id)(MgNote*, MgInt*);
    MgResult (MG_CALL *note_get_info)(MgNote*, MgNoteInfo*);
    MgResult (MG_CALL *note_set_info)(MgNote*, const MgNoteInfo*);
    MgResult (MG_CALL *note_get_children_count)(MgNote*, MgInt*);
    MgResult (MG_CALL *note_get_child)(MgNote*, MgInt index, MgNote**);
    MgResult (MG_CALL *note_get_parent)(MgNote*, MgNote**);
    MgResult (MG_CALL *note_append_child)(MgNote*, MgNote* child);
    MgResult (MG_CALL *note_delete_child)(MgNote*, MgNote* child);
    MgResult (MG_CALL *note_clone)(MgNote*, MgNote**);
    MgResult (MG_CALL *note_replace_with)(MgNote*, const MgNote* source, MgBool requires_sort);
    MgResult (MG_CALL *note_copy_info_to)(MgNote*, MgNote* destination);
    MgResult (MG_CALL *note_get_base_note)(MgNote*, MgNote**);
    MgResult (MG_CALL *note_offset_child)(MgNote*, MgInt offset_tick);
    MgResult (MG_CALL *note_flip_h)(MgNote*, MgBool recursive);

    MgResult (MG_CALL *chart_create_event)(MgChart*, MgEventKind, MgEvent**);
    MgResult (MG_CALL *chart_append_event)(MgChart*, MgEvent*);
    MgResult (MG_CALL *chart_delete_event)(MgChart*, MgEvent*);
    MgResult (MG_CALL *chart_find_event_timeline_speed)(MgChart*, MgInt tick, MgInt timeline_id, MgEvent**);
    MgResult (MG_CALL *chart_find_event_note_speed_modifier)(MgChart*, MgInt tick, MgEvent**);
    MgResult (MG_CALL *chart_find_event_bpm)(MgChart*, MgInt tick, MgEvent**);
    MgResult (MG_CALL *chart_find_event_beat_change)(MgChart*, MgInt bar, MgEvent**);

    MgResult (MG_CALL *event_get_kind)(MgEvent*, MgEventKind*);
    MgResult (MG_CALL *event_get_id)(MgEvent*, MgInt*);
    MgResult (MG_CALL *event_timeline_speed_get_info)(MgEvent*, MgEventTimelineSpeedInfo*);
    MgResult (MG_CALL *event_timeline_speed_set_info)(MgEvent*, const MgEventTimelineSpeedInfo*);
    MgResult (MG_CALL *event_note_speed_modifier_get_info)(MgEvent*, MgEventNoteSpeedModifierInfo*);
    MgResult (MG_CALL *event_note_speed_modifier_set_info)(MgEvent*, const MgEventNoteSpeedModifierInfo*);
    MgResult (MG_CALL *event_bpm_get_info)(MgEvent*, MgEventBpmInfo*);
    MgResult (MG_CALL *event_bpm_set_info)(MgEvent*, const MgEventBpmInfo*);
    MgResult (MG_CALL *event_beat_change_get_info)(MgEvent*, MgEventBeatChangeInfo*);
    MgResult (MG_CALL *event_beat_change_set_info)(MgEvent*, const MgEventBeatChangeInfo*);
    MgResult (MG_CALL *event_replace_with)(MgEvent*, const MgEvent* source);
    MgResult (MG_CALL *event_copy_info_to)(MgEvent*, MgEvent* destination);
} MgHostApi;
```

型別event get/set、`event_replace_with`、`event_copy_info_to` はkind一致を必須とし、不一致なら `MG_ERR_INVALID_ARGUMENT` を返す。

## 15. Undo規約

編集をまとめる場合は `beginRecording` の成功後に変更し、成功時は `commitRecording`、失敗時は `discardRecording` を呼ぶ。

`discardRecording` が既に行った編集をrollbackするとは規定しない。公式SDKとサンプルは、discard、nested recording、失敗時の状態を契約として説明していないためである。ABI実装はSDKの戻り値をそのまま成功・失敗へ変換する。

`invoke` 復帰時にrecording中のままにしてはならない。

## 16. Margrete SDKとの対応

ABI 1は基準SDKの全公開methodへ次の方針で対応する。

- `addRef` / `release`: `object_add_ref` / `object_release`
- `queryInterface`: 型付きhandle、event kind、型別create/findで代替
- Undo全method: `undo_*`
- Context全method: `context_*`
- Document全method: `document_*`
- Chart全method: `chart_*`
- Note全method: `note_*`
- Event base全method: `event_get_id` と `event_get_kind`
- 4 event interfaceのget/set/replace/copy: `event_*`
- Command全method: `MgPluginApi` のcommand callback

新しい公式SDKへ追従する際は完全対応表を更新し、未対応methodがある状態で安定ABI releaseを行ってはならない。

## 17. 実装上のSDK補正

基準SDKヘッダの `MpGuid::operator==` は `sizeof(this)` を使用し、`operator!=` は自己再帰する実装になっている。marmkmtはこれらへ依存せず、GUIDの16byteすべてを明示比較する。

SDK C++ ABIを再実装してはならない。公式ヘッダを使用し、C++ SDKとの境界はmarmkmt内部に限定する。

## 18. 検証要件

ABI 1を凍結する前に次を満たさなければならない。

- 正規Cヘッダとstatic libraryを用意する。
- 公開構造体のsize、alignment、全offsetをC/C++とwrapperで照合する。
- 全Host APIのNULL、型違い、解放済み、誤thread、範囲外試験を行う。
- 偽Hostと偽PluginによるABI conformance testを用意する。
- SDK定数、構造体、GUID、virtual method順序の差分をCIで検出する。
- 公式サンプル相当のnote、child note、BPM event、Undo、updateをMargrete実機で確認する。
- Undo全method、`findEvent*`、event replace/copyを実機確認する。
- commandの同時invokeと再入を拒否することを確認する。
- 捕捉可能な例外/panicがcommandをpoisonし、Margreteを継続利用できることを確認する。
- 通常時と失敗時のどちらでもファイルログを生成しないことを確認する。

## 19. 上流契約が不明な挙動

次は公式ヘッダと公式サンプルだけでは契約を確定できない。安全側の規約を適用したうえで、調査pluginによる実測または上流確認を行う。

- 各out `ppobj` の参照count。確認までは所有参照として扱う。
- appendが入力参照を保持するか。呼び出し側の参照は常に独立してreleaseする。
- delete後に呼び出し側handleをreadできるか。確認まではdelete後のreadを禁止し、releaseだけを許可する。
- parent、child、chart間の内部循環参照。
- `discardRecording`、nested recording、失敗時のUndo状態。
- plugin export、command methodの呼び出し順、回数、thread、再入。
- commandのrelease時期と最終plugin DLLのunload時期。
- `getCommandName` にMargreteが渡す実際のbuffer容量。
- SDK objectのthread affinity。
- `MP_SDK_VERSION` とvirtual tableの上流互換性規則。

これらの実測結果によってABIの安全制約を緩和してはならない。緩和が必要な場合はexperimental期間中に仕様を改訂し、ABI 1凍結前に完了する。

## 20. ライセンス

marmkmtの配布前に、公式SDK由来の定数、構造体、GUID、ヘッダ利用についてMIT Licenseの表示要件を満たす。正規Cヘッダと配布物に必要なcopyright noticeおよびlicense noticeを含める。
