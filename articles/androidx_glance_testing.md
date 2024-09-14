---
title: "【Android】Glance 単体テスト"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Android", "Glance", "ウィジェット", "Test"]
published: false
---

本記事が前提とする Glance の導入・状態管理は別記事で紹介しています

https://qiita.com/Seo-4d696b75/items/235967ed0c4332683f6e

---------

# 安定版 glance*-testing がリリース🎉

https://developer.android.com/jetpack/androidx/releases/glance#1.1.0

1.0.0 までは[glance-experimental-tools](https://github.com/google/glance-experimental-tools)という別リポジトリで試験的に提供されていましたが、ついに安定版として androidx に仲間入りしました！

# セットアップ

Android SDK を単体テストで利用するため Robolectric を併用します

```diff gradle
 android {
+    testOptions {
+        unitTests {
+            isIncludeAndroidResources = true
+        }
+    }
 }

 dependencies {
+    testImplementation("junit:junit:4.13.2")
+    testImplementation("androidx.test.ext:junit:1.2.1")
+    testImplementation("org.robolectric:robolectric:4.13")
+    testImplementation("androidx.glance:glance-appwidget-testing:1.1.0")
 }
```

# テスト

Jetpack Compose のテストと同様に書けます！

```kotlin
@RunWith(AndroidJUnit4::class)
@Config(sdk = [34])
class CounterWidgetTest {
    @Test
    fun CounterScreen_idle() = runGlanceAppWidgetUnitTest {
        // 1. 描画サイズの指定
        setAppWidgetSize(DpSize(100.dp, 100.dp))

        // （必要なら）Contextの用意
        val context: Context = ApplicationProvider.getApplicationContext()

        // 2. Glance の指定
        provideComposable {
            CompositionLocalProvider(
                LocalContext provides context
            ) {
                WidgetTheme {
                    CounterScreen(
                        count = 10,
                        isLoading = false,
                        onIncrement = { },
                        onDecrement = { },
                    )
                }
            }
        }

        // 3. 検査の実行
        onNode(hasText("10")).assertExists()
    }
}
```

:::message
テストが容易な設計を心掛ける

Glanceの状態を参照する`currentState()`はトップレベルに置き、各コンポーネントの状態は引数として外部から簡単に指定できるようにします
:::

