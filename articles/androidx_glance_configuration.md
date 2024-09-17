---
title: "【Android】Glance ウィジェットの設定・再設定に対応する"
emoji: "⚙️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Android", "Glance", "ウィジェット"]
published: true
publication_name: "yumemi_inc"
---

本記事が前提とする Glance の導入・状態管理は別記事で紹介しています

https://qiita.com/Seo-4d696b75/items/235967ed0c4332683f6e

---

# ウィジェットの設定・再設定

ウィジェットを追加する時、もしくは既に追加されたウィジェットを長押し（Android 12 以降のみ）すると設定画面を開くことができます。

:::message
添付動画は Pixel のエミュレータ（Android 13）で撮影しました。Android バージョン・端末メーカーによっては見た目が異なる場合もあります。
:::

| 初回の設定                                                                     | 再設定                                                                         |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| ![](https://storage.googleapis.com/zenn-user-upload/22da772c95df-20240914.gif) | ![](https://storage.googleapis.com/zenn-user-upload/eaa71f177709-20240914.gif) |

# 実装

基本的な実装方法は公式ドキュメントで説明されています。
ただし `RemoteView` を用いた従来のウィジェット実装を前提としており、Glance の場合は一部で異なる対応が必要です。

https://developer.android.com/develop/ui/views/appwidgets/configuration

本記事で実装したデモアプリ＆ウィジェットは GitHub で公開しています

https://github.com/Seo-4d696b75/glance-widget-demo

## ウィジェットの設定の種類と指定方法

Android バージョンによって異なります

| 設定方法                  | 表示のタイミング                                               | Android 12 未満 |     Android 12 以降     |
| :------------------------ | :------------------------------------------------------------- | :-------------: | :---------------------: |
| 初回設定（Configuration） | ウィジェットを Home 画面に新規追加したとき（デフォルトで有効） |       ⭕        | ⭕ <br>（スキップ可能） |
| 再設定（Reconfiguration） | 追加済みのウィジェットを長押しする（追加指定が必要）           |       ❌        |           ⭕            |

Android 12 以降ではウィジェットの設定ファイルに`android:widgetFeatures`属性を追加して挙動を指定可能です。

| 値                     | 説明                                                 |
| :--------------------- | :--------------------------------------------------- |
| reconfigurable         | 再設定を可能にする（デフォルトでは無効化されている） |
| configuration_optional | ウィジェット追加時の設定をスキップする               |

（例）初回の設定をスキップ＆再設定を可能にする

```diff xml:src/res/xml-v28/appwidget_info.xml
 <?xml version="1.0" encoding="utf-8"?>
 <appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
     android:description="@string/widget_description"
     android:initialLayout="@layout/glance_default_loading_layout"
     android:minWidth="110dp"
     android:minHeight="40dp"
     android:previewImage="@null"
     android:resizeMode="none"
     android:updatePeriodMillis="1800000"
-    android:widgetCategory="home_screen"/>
+    android:widgetCategory="home_screen"
+    android:widgetFeatures="reconfigurable|configuration_optional"/>
```

:::message alert
`android:widgetFeatures`属性の指定は Android 9 (API 28) 以降で可能ですが、実際に効力を発揮するのは Android 12 以降のみです
:::

## 設定画面の Activity を追加する

`AppWidgetManager.EXTRA_APPWIDGET_ID`という key で設定対象のウィジェットの識別子を参照できます。

また`setResult()`で設定の成功可否をシステム側に伝える必要があるのですが、
設定用の Activity が途中で kill される場合も考慮して `onCreate()`のタイミングでとりあえず`RESULT_CANCELED`をセットしておくよう推奨されています。

```kotlin:CounterConfigureActivity.kt
class CounterConfigureActivity : ComponentActivity() {

    private val appWidgetId: Int by lazy {
        intent?.extras?.getInt(
            AppWidgetManager.EXTRA_APPWIDGET_ID,
            AppWidgetManager.INVALID_APPWIDGET_ID,
        ) ?: AppWidgetManager.INVALID_APPWIDGET_ID
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setResult(
            RESULT_CANCELED,
            Intent().apply {
                putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetId)
            },
        )

        setContent {
            ConfigureScreen(
                onCompleted = { /* TODO */ },
            )
        }
    }
}
```

マニフェストに Activity を宣言するときは、特別な intent-filter を指定する必要があります。

```diff xml:AndroidManifest.xml
 <?xml version="1.0" encoding="utf-8"?>
 <manifest xmlns:android="http://schemas.android.com/apk/res/android"
     xmlns:tools="http://schemas.android.com/tools">

     <application>
+        <activity
+            android:name=".CounterConfigureActivity"
+            android:exported="true">
+            <intent-filter>
+                <action android:name="android.appwidget.action.APPWIDGET_CONFIGURE" />
+            </intent-filter>
+        </activity>
     </application>
 </manifest>
```

同時にウィジェット側の設定ファイルでも、どの Activity を使用するか指定してあげます。

```diff xml:res/xml*/appwidget_info.xml
 <?xml version="1.0" encoding="utf-8"?>
 <appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
+    android:configure="com.seo4d696b75.android.glance_widget_demo.CounterConfigureActivity"
     android:description="@string/widget_description"
     android:initialLayout="@layout/glance_default_loading_layout"
     android:minWidth="110dp"
     android:minHeight="40dp"
     android:previewImage="@null"
     android:resizeMode="none"
     android:updatePeriodMillis="1800000"
     android:widgetCategory="home_screen" />
```

## 設定値の反映

設定画面で入力 or 選択された値をウィジェットに反映する必要があります。
まずは`CounterWidget`に関数を用意します。

```diff kotlin:CounterWidget.kt
 class CounterWidget : GlanceAppWidget() {

+    // 再設定時に現在の状態を参照する
+    suspend fun getCount(context: Context, glanceId: GlanceId): Int {
+        val pref: Preferences = getAppWidgetState(context, glanceId)
+        return pref[PREF_KEY_COUNT] ?: 0
+    }

+    // 設定完了時に状態を反映＆描画する
+    suspend fun setCount(context: Context, glanceId: GlanceId, count: Int) {
+        updateAppWidgetState(context, glanceId) {
+            it[PREF_KEY_COUNT] = count
+        }
+        update(context, glanceId)
+    }

    companion object {
        private val PREF_KEY_COUNT = intPreferencesKey("pref_key_count")
    }
}
```

`CounterWidget`の関数を直接呼び出しても問題ないのですが、デモアプリでは module を分割している影響で間に適当な抽象化を挟んでいます。

:::message
設定画面の Activity に渡されるウィジェット識別子は従来の`RemoteView`実装と同じ`Int`型であるため、`GlanceId`への変換が必要な点に注意してください。
:::

```kotlin:CounterWidgetMediator.kt
// module分割の仕方によってはインターフェイス定義・実装クラスが分かれる場合もあります
class CounterWidgetMediator @Inject constructor(
    @ApplicationContext
    private val context: Context,
) {
    private val glanceAppWidgetManager = GlanceAppWidgetManager(context)
    private val widget = CounterWidget()

    suspend fun getCount(appWidgetId: Int): Int {
        val glanceId = glanceAppWidgetManager.getGlanceIdBy(appWidgetId)
        return widget.getCount(context, glanceId)
    }

    suspend fun setCount(appWidgetId: Int, count: Int) {
        val glanceId = glanceAppWidgetManager.getGlanceIdBy(appWidgetId)
        widget.setCount(context, glanceId, count)
    }
}
```

後は`CounterWidgetMediator`を UI 側に DI して呼び出すだけです。設定画面の UI 実装（Jetpack Compose）＆状態管理（ViewModel）は本記事の範囲外のため説明を割愛します。

最後に`setResult(RESULT_OK)`を忘れずに呼び出しましょう。

```diff kotlin:CounterConfigureActivity.kt
 class CounterConfigureActivity : ComponentActivity() {

     override fun onCreate(savedInstanceState: Bundle?) {
         super.onCreate(savedInstanceState)

         setResult(
             RESULT_CANCELED,
             Intent().apply {
                 putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetId)
       　     },
         )

         setContent {
             ConfigureScreen(
                 onCompleted = {
+                    setResult(
+                        RESULT_OK,
+                        Intent().apply {
+                            putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetId)
+                        },
+                    )
+                    finish()
                 },
             )
         }
     }
 }
```
