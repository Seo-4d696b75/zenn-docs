---
title: "【Android】アプリ内からウィジェットを追加する"
emoji: "➕️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Android", "Glance", "ウィジェット"]
published: true
publication_name: "yumemi_inc"
---

# ホーム画面から追加する方法と問題点

独自の実装不要でもウィジェットを追加する方法はあります。しかし操作の手順が少々手間であり、ユーザーがウィジェットを追加しようと明確な意思が無いとたどり着けない点が辛いです。アプリ外でもユーザーに積極的に訴求できるのがウィジェットの最大の利点ですが、ウィジェットをユーザーに追加してもらうよう訴求するのが難しいのでは本末転倒です...

| ホーム画面の設定 | アプリのショートカット |  
|:-----:|:------:|  
| ホーム画面を長押ししてウィジェットを選ぶ | ホーム画面上のアプリアイコンを長押しする |  
|![](https://storage.googleapis.com/zenn-user-upload/772c6b0848e7-20240922.gif)|![](https://storage.googleapis.com/zenn-user-upload/4b9d3b19390d-20240922.gif)|  

:::message
添付動画は Pixel のエミュレータ（Android 14）で撮影しました。Android バージョン・端末メーカーによっては見た目が異なる場合もあります。
:::

# アプリ内からウィジェットを追加する

そこで今回はアプリ内からウィジェットを追加する方法を実装します。独自実装が必要になりますが、自由な動線でユーザーにウィジェット追加を訴求できるのが強みになっています。

| 設定なし | 設定あり |  
|:-----:|:-----:|  
|![](https://storage.googleapis.com/zenn-user-upload/f9cea1f8cfba-20240922.gif)|![](https://storage.googleapis.com/zenn-user-upload/57689dc2e32d-20240922.gif)|  

本記事で実装したデモアプリ＆ウィジェットは GitHub で公開しています

https://github.com/Seo-4d696b75/glance-widget-demo

## 実装

使用するAPIの詳細は公式ドキュメントを参照してください

https://developer.android.com/develop/ui/views/appwidgets/discoverability

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            YourScreen(
                addWidget = ::addWidget,
            )
        }
    }

    /**
     * ウィジェットを追加する
     * 
     * @param configure 設定画面を表示するかフラグ
     */
    private fun addWidget(configure: Boolean) {
        val manager = AppWidgetManager.getInstance(this)
        if (manager.isRequestPinAppWidgetSupported) {
            // AppWidgetProvider もしくは GlanceAppWidgetReceiver を指定
            val provider = ComponentName(this, YourWidgetReceiver::class.java)
            val callback = if (configure) {
                // 追加に成功したらウィジェット設定用のActivityを起動する
                PendingIntent.getActivity(
                    this,
                    0,
                    Intent(this, YourConfigureActivity::class.java),
                    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE,
                )
            } else {
                // 追加成功時のコールバックは未指定でも可
                null
            }
            manager.requestPinAppWidget(provider, null, callback)
        }
    }
}
```

コールバックを指定する＆追加に成功した場合はウィジェットの識別子を`AppWidgetManager.EXTRA_APPWIDGET_ID`で参照できます。

:::message alert
Android 12 以降では `PendingIntent.FLAG_*MUTABLE` の指定が必須ですが、Intentへ追加される `AppWidgetManager.EXTRA_APPWIDGET_ID` を利用するには `PendingIntent.FLAG_MUTABLE` を選択してください。
:::

```kotlin
@AndroidEntryPoint
class YourConfigureActivity : ComponentActivity() {
    private val appWidgetId: Int = requireNotNull(
        intent?.extras?.getInt(
            AppWidgetManager.EXTRA_APPWIDGET_ID,
            AppWidgetManager.INVALID_APPWIDGET_ID,
        )
    )
}
```

もしくは ViewModel で受け取ることもできます

```kotlin
@HiltViewModel
class ConfigureViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
) : ViewModel() {

    private val appWidgetId: Int =
        requireNotNull(savedStateHandle[AppWidgetManager.EXTRA_APPWIDGET_ID])
}
```
