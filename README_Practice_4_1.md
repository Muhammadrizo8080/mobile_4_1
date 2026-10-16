# KMP Practice 4.1 — 30 ta topshiriq javoblari

**Part A** — What is Kotlin Multiplatform (1–15)
**Part B** — KMP Project Configuration (16–30)

> Eslatma: barqarorlik darajalari (Task 11) bo'yicha ma'lumot KMP rasmiy hujjatlariga asoslangan. Ma'ruza slaydlaridagi formulirovka bilan solishtirib ko'ring.

## Mundarija

- [Part A (Task 1–15)](#part-a--what-is-kotlin-multiplatform-115)
- [Part B (Task 16–30)](#part-b--kmp-project-configuration-1630)

---

# Part A — What is Kotlin Multiplatform (1–15)

## Task 1 — KMP vs Flutter vs React Native

| Texnologiya | Qanday ishlaydi |
|---|---|
| **Flutter** | UI ni **o'zining render dvigateli** (Skia/Impeller) orqali chizadi. Native vidjetlardan foydalanmaydi, piksellarni o'zi chizadi. Kod Dart tilida. |
| **React Native** | JavaScript kodi JS dvigatelida (Hermes) ishlaydi va **JS ↔ native ko'prik (bridge/JSI) orqali native view'larga** murojaat qiladi. |
| **KMP** | Umumiy Kotlin kodi **har bir platforma uchun o'z native ko'rinishiga kompilyatsiya bo'ladi**. Ko'prik ham, alohida virtual mashina ham yo'q. UI native qolishi mumkin (Jetpack Compose / SwiftUI). |

iOS'da umumiy Kotlin kodi **Kotlin/Native (LLVM asosida) orqali native mashina kodiga kompilyatsiya qilinib, Objective-C/Swift ko'ra oladigan framework** bo'ladi. Android'da esa oddiy JVM bytecode.

---

## Task 2 — Target va Kotlin yetib borish usuli

| Target | Kotlin qanday yetib boradi |
|---|---|
| Android | **Kotlin/JVM** (bytecode, ART'da ishlaydi) |
| iOS | **Kotlin/Native** (native binary/framework) |
| Browser (1-variant) | **Kotlin/JS** (JavaScript'ga kompilyatsiya; Stable) |
| Browser (2-variant) | **Kotlin/Wasm** (WebAssembly'ga kompilyatsiya; Beta) |
| Desktop | **Kotlin/JVM** (masalan, Compose Desktop) |
| Server | **Kotlin/JVM** (Ktor, Spring Boot) |

**iOS uchun kompilyator texnologiyasi:** Kotlin/Native, LLVM backend.

---

## Task 3 — Shared yoki Native

| Element | Qaror | Bir qatorli asos |
|---|---|---|
| Ktor API client | **Shared (commonMain)** | Ktor multiplatform; so'rov mantig'i bir xil, faqat engine platformaga xos. |
| Biometric login | **Native** | `BiometricPrompt` / `LocalAuthentication` platforma API'lari; common'da faqat interfeys bo'ladi. |
| Input validation | **Shared** | Sof mantiq, platformaga bog'liq emas. |
| Push notifications | **Native** | FCM/APNs OS darajasida ro'yxatdan o'tadi; common'da faqat payload modeli bo'lishi mumkin. |
| SQLDelight storage | **Shared** | SQLDelight multiplatform; faqat driver platformaga xos. |
| Deep-link navigation | **Native** | Havolani OS ushlaydi va navigatsiya UI'ga bog'liq; common'da faqat URL → route parser bo'lishi mumkin. |

---

## Task 4 — expect/actual: Platform

`actual` deklaratsiyalar `expect` bilan **bir xil package**da bo'lishi kerak.

**commonMain** (berilgan)

```kotlin
interface Platform { val name: String }

expect fun getPlatform(): Platform
```

**androidMain**

```kotlin
import android.os.Build

class AndroidPlatform : Platform {
    override val name: String = "Android ${Build.VERSION.SDK_INT}"
}

actual fun getPlatform(): Platform = AndroidPlatform()
```

**iosMain**

```kotlin
import platform.UIKit.UIDevice

class IOSPlatform : Platform {
    override val name: String =
        UIDevice.currentDevice.systemName() + " " + UIDevice.currentDevice.systemVersion
}

actual fun getPlatform(): Platform = IOSPlatform()
```

**jvmMain** (desktop)

```kotlin
class JVMPlatform : Platform {
    override val name: String = "Java ${System.getProperty("java.version")}"
}

actual fun getPlatform(): Platform = JVMPlatform()
```

---

## Task 5 — Xatoni topish

```kotlin
// commonMain
expect fun currentTimeMillis(): Long {
    return 0L
}
```

**Xato:** `expect` deklaratsiyasida **tana (body) bo'lishi mumkin emas**. U faqat "shartnoma", implementatsiya `actual`'da yoziladi.

**To'g'ri variant:**

```kotlin
// commonMain
expect fun currentTimeMillis(): Long

// androidMain va jvmMain
actual fun currentTimeMillis(): Long = System.currentTimeMillis()

// iosMain
import platform.Foundation.NSDate
import platform.Foundation.timeIntervalSince1970

actual fun currentTimeMillis(): Long =
    (NSDate().timeIntervalSince1970 * 1000).toLong()
```

Agar umumiy standart implementatsiya kerak bo'lsa, `expect` ishlatmasdan oddiy funksiya yoki interfeys yozing.

---

## Task 6 — Yetishmayotgan `actual` (iOS)

**Nima buziladi:** `iosArm64()` va `iosSimulatorArm64()` target'lari uchun kompilyatsiya **xato bilan to'xtaydi**: *"Expected declaration has no actual declaration in module ... for iosArm64 / iosSimulatorArm64"*. `expect`'ga har bir target'da mos `actual` bo'lishi shart.

**Ikkita yechim:**

1. `actual`'ni **`iosMain`**'ga qo'shish. U ikkala iOS target uchun ham umumiy:

```kotlin
// iosMain
actual fun getPlatform(): Platform = IOSPlatform()
```

   Yoki alohida `iosArm64Main` va `iosSimulatorArm64Main`'ga yozish (ortiqcha takrorlanish).

2. `expect/actual`'dan voz kechib, **interfeys + injeksiya** ishlatish: `commonMain`'da interfeys, har bir platforma o'z implementatsiyasini ilovaga uzatadi. Yoki iOS target'larini loyihadan olib tashlash (iOS kerak bo'lmasa).

---

## Task 7 — `iosArm64` uchun kompilyator birlashtiradigan source set'lar

Default hierarchy template bo'yicha zanjir:

```
commonMain → nativeMain → appleMain → iosMain → iosArm64Main
```

**Nega `appleMain` kodi iOS target'larida ko'rinadi, lekin `androidMain`'da yo'q:**
Ko'rinish faqat `dependsOn` zanjiri bo'ylab **yuqoridan pastga** oqadi. `iosMain` `appleMain`'ga, u esa `nativeMain`'ga bog'langan, shuning uchun iOS target'lari `appleMain` e'lonlarini ko'radi. `androidMain` esa boshqa tarmoqda turadi (`commonMain → androidMain`), unda `appleMain` yo'q. Bundan tashqari `appleMain` Apple platform API'lariga (`platform.Foundation`, `UIKit`) ega, ular Android'da mavjud emas.

---

## Task 8 — `iosMain`: `systemVersion`

```kotlin
// iosMain
import platform.UIKit.UIDevice

fun iosSystemVersion(): String = UIDevice.currentDevice.systemVersion
```

**Nega `commonMain`'da bo'lmaydi:** `platform.UIKit` Apple Kotlin/Native target'lari uchun mavjud bo'lgan platforma kutubxonasi. `commonMain` Android, JVM, JS kabi boshqa target'larga ham kompilyatsiya bo'ladi, u yerda UIKit yo'q, shuning uchun `Unresolved reference` xatosi chiqadi. Kerak bo'lsa, `expect/actual` bilan umumiy interfeys beriladi.

---

## Task 9 — Ktor `sourceSets`

```kotlin
val ktorVersion = "3.0.3"   // o'z loyihangizdagi versiyaga almashtiring

kotlin {
    androidTarget()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        commonMain.dependencies {
            implementation("io.ktor:ktor-client-core:$ktorVersion")
        }
        androidMain.dependencies {
            implementation("io.ktor:ktor-client-okhttp:$ktorVersion")
        }
        iosMain.dependencies {
            implementation("io.ktor:ktor-client-darwin:$ktorVersion")
        }
    }
}
```

---

## Task 10 — KMP va Compose Multiplatform

- **Kotlin Multiplatform (KMP)** — kodni platformalar o'rtasida ulashish texnologiyasi (til + kompilyator + Gradle plagin). Odatda **mantiq** (network, ma'lumotlar, biznes qoidalar) ulashiladi.
- **Compose Multiplatform (CMP)** — JetBrains'ning Jetpack Compose asosidagi **UI framework'i**, UI'ni Android, iOS, desktop va web'da ulashadi.

**Qaysi biri boshqasisiz ishlaydi:** KMP CMP'siz ishlaydi (UI native qoladi). CMP esa KMP'ga **tayanadi**, KMP'siz ishlay olmaydi.

**Mantiq KMP, UI native bo'lgan modul tuzilmasi:**

```
MyApp/
├── shared/            ← KMP kutubxona (commonMain, androidMain, iosMain)
│   └── iOS framework "Shared" chiqaradi
├── androidApp/        ← Jetpack Compose UI, implementation(project(":shared"))
└── iosApp/            ← Xcode loyiha, SwiftUI UI, Shared.framework'ni ishlatadi
```

---

## Task 11 — Barqarorlik darajalari va production qarori

| Komponent | Daraja | Production'da ishlatamanmi? |
|---|---|---|
| **iOS** (KMP core, Kotlin/Native) | **Stable** (2023-yildan) | **Ha.** Barqaror, ko'plab kompaniyalar ishlatadi. |
| **Kotlin/Wasm** | **Beta** | **Ehtiyotkorlik bilan.** API va tooling o'zgarishi mumkin; muhim loyihada Kotlin/JS (Stable) xavfsizroq. |
| **expect/actual classes** | **Beta** (funksiya/property'lar esa Stable) | **Ha, ehtiyot bilan.** Kompilyator ogohlantirish beradi; oddiy interfeys afzal, expect/actual faqat platforma API'lari uchun. |
| **Compose Multiplatform for web** | **Beta** (Wasm asosida) | **Hozircha yo'q / kichik loyihalarda.** Ichki vositalar uchun mumkin, jiddiy production uchun Stable emas. |

> Compose Multiplatform for iOS esa Stable (2025-yildan), shuning uchun iOS UI'ni CMP bilan ulashish production'ga yaroqli.

---

## Task 12 — `commonTest` va Gradle task'lar

**commonMain**

```kotlin
fun isValidEmail(text: String): Boolean =
    Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$").matches(text)
```

**commonTest** (`shared/src/commonTest/kotlin/EmailValidatorTest.kt`)

```kotlin
import kotlin.test.Test
import kotlin.test.assertFalse
import kotlin.test.assertTrue

class EmailValidatorTest {
    @Test
    fun validEmail() = assertTrue(isValidEmail("user@example.com"))

    @Test
    fun invalidEmail() = assertFalse(isValidEmail("not-an-email"))
}
```

`build.gradle.kts`'da test kutubxonasi:

```kotlin
commonTest.dependencies { implementation(kotlin("test")) }
```

**Gradle task'lar:**

```bash
./gradlew :shared:jvmTest                  # JVM'da
./gradlew :shared:iosSimulatorArm64Test    # iOS simulyatorda (faqat macOS)
./gradlew :shared:allTests                 # barcha target'lar
```

---

## Task 13 — KMP'ning ikkita cheklovi (2026) va yechimlar

**1. Swift interop.** Kotlin kodi Objective-C header orqali Swift'ga ochiladi. Natijada generic'lar, sealed class'lar, `suspend`/`Flow`, enum'lar Swift'da noqulay ko'rinadi.
*Yechim:* **SKIE** yoki **KMP-NativeCoroutines** kabi vositalardan foydalanish va ochiq API'ni Swift'ga mos, sodda saqlash (oddiy data class, callback/async wrapper'lar).

**2. Binary hajmi.** Kotlin/Native runtime va kutubxonalar iOS ilovasiga qo'shiladi (taxminan bir necha MB), bir nechta framework bo'lsa ular takrorlanadi.
*Yechim:* bitta **umbrella framework** (static), release optimizatsiyalari va keraksiz bog'liqliklarni olib tashlash.

*(Qo'shimcha cheklov: jamoa ko'nikmalari.)* iOS dasturchilar Kotlin/Gradle'ga ega bo'lishi kerak. Yechim: kichik modul (masalan, validatsiya) bilan boshlash va o'qitish.

---

## Task 14 — `LoginValidator`

**commonMain** — `shared/src/commonMain/kotlin/uz/ttpu/shared/auth/LoginValidator.kt`

```kotlin
package uz.ttpu.shared.auth

enum class EmailError { EMPTY, INVALID_FORMAT }
enum class PasswordError { TOO_SHORT, NO_DIGIT, NO_UPPERCASE }

data class LoginValidationResult(
    val emailError: EmailError?,
    val passwordError: PasswordError?
) {
    val isValid: Boolean get() = emailError == null && passwordError == null
}

class LoginValidator {
    private val emailRegex =
        Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")

    fun validate(email: String, password: String): LoginValidationResult {
        val emailError = when {
            email.isBlank() -> EmailError.EMPTY
            !emailRegex.matches(email) -> EmailError.INVALID_FORMAT
            else -> null
        }
        val passwordError = when {
            password.length < 8 -> PasswordError.TOO_SHORT
            password.none { it.isDigit() } -> PasswordError.NO_DIGIT
            password.none { it.isUpperCase() } -> PasswordError.NO_UPPERCASE
            else -> null
        }
        return LoginValidationResult(emailError, passwordError)
    }
}
```

**Android ViewModel** (`androidApp`)

```kotlin
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import uz.ttpu.shared.auth.LoginValidationResult
import uz.ttpu.shared.auth.LoginValidator

class LoginViewModel(
    private val validator: LoginValidator = LoginValidator()
) : ViewModel() {

    private val _result = MutableStateFlow<LoginValidationResult?>(null)
    val result: StateFlow<LoginValidationResult?> = _result

    fun onSubmit(email: String, password: String) {
        _result.value = validator.validate(email, password)
    }
}
```

**SwiftUI view model** (`iosApp`)

```swift
import SwiftUI
import Shared

@MainActor
final class LoginViewModel: ObservableObject {
    private let validator = LoginValidator()

    @Published var emailMessage: String? = nil
    @Published var passwordMessage: String? = nil

    func submit(email: String, password: String) {
        let result = validator.validate(email: email, password: password)

        emailMessage = result.emailError == nil ? nil : "Email noto'g'ri"
        passwordMessage = result.passwordError == nil ? nil : "Parol talabga javob bermaydi"

        if result.isValid {
            // login so'rovini yuborish
        }
    }
}
```

Validatsiya qoidalari bir marta yoziladi, ikkala ilova bir xil natija beradi.

---

## Task 15 — Android-only ilovani iOS'ga ko'chirish tartibi

**Tartib:**

1. **Data models**
2. **Business rules**
3. **Networking**
4. **Local storage**
5. **UI** (oxirgi, ixtiyoriy)

**Nega birinchi bo'lib data models:**

- Platforma API'lariga bog'liq emas, sof Kotlin (`kotlinx.serialization` bilan).
- Android kod allaqachon Kotlin'da, fayllarni deyarli o'zgarishsiz ko'chirish mumkin, xavf past.
- Qolgan barcha qatlamlar (qoidalar, network, storage) modellarga tayanadi.
- Birinchi ko'chirish `shared` modul va iOS framework yig'ish zanjirini sinab ko'rishga imkon beradi, kam xavf bilan "tez g'alaba" beradi.

Keyin biznes qoidalar (sof mantiq), so'ng Ktor/network, so'ng SQLDelight, UI esa eng oxirida va faqat Compose Multiplatform kerak bo'lsa ko'chiriladi.

---

# Part B — KMP Project Configuration (16–30)

## Task 16 — Bitta modul vs bir nechta modul

| | Afzallik | Kamchilik |
|---|---|---|
| **Bitta shared modul** | Sozlash oddiy, iOS uchun bitta framework (umbrella kerak emas). | Kod o'sgani sari build sekinlashadi, chegaralar yo'q. |
| **Bir nechta shared modul** | Inkapsulyatsiya, parallel/inkremental build, jamoalar o'z modullariga egalik qiladi. | Konfiguratsiya murakkabroq, iOS uchun umbrella modul shart. |

**Bo'lish signali:** build vaqti sezilarli oshib ketganda, bir-biriga bog'liq bo'lmagan funksiyalar ham birga qayta kompilyatsiya bo'lganda, jamoalar bir-biriga halaqit bera boshlaganda (merge-konfliktlar) yoki ba'zi qismni mustaqil qayta ishlatish kerak bo'lganda.

---

## Task 17 — `settings.gradle.kts` va bog'liqliklar

**settings.gradle.kts**

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "MyApp"

include(":shared")
include(":androidApp")
include(":desktopApp")
```

**androidApp/build.gradle.kts**

```kotlin
dependencies {
    implementation(project(":shared"))
}
```

**desktopApp/build.gradle.kts** — oddiy JVM modul bo'lsa:

```kotlin
dependencies {
    implementation(project(":shared"))
}
```

Agar `desktopApp` ham KMP modul (jvm target) bo'lsa:

```kotlin
kotlin {
    jvm()
    sourceSets {
        jvmMain.dependencies {
            implementation(project(":shared"))
        }
    }
}
```

---

## Task 18 — Ikkita feature modul = ikkita iOS framework

**Ilova ichida nima bo'ladi:** har bir framework o'zi ishlatadigan **barcha Kotlin bog'liqliklarini o'z ichiga oladi**. Umumiy kutubxona (masalan, `:core` yoki networking kutubxonasi) ilovada **ikki nusxada** bo'ladi, har biri o'zining Kotlin runtime nusxasi bilan.

**Aniq buzilish (shared state):** kutubxonada `object TokenStore` (singleton) yoki umumiy kesh/`StateFlow` bor deylik. `FeatureAuth` framework orqali login qilib token yozsangiz, `FeatureProfile` framework **o'zining alohida `TokenStore` nusxasini** ko'radi va u bo'sh, ya'ni foydalanuvchi "login qilmagan". Bitta deb o'ylagan holat aslida ikki mustaqil nusxa. Qo'shimcha: Swift'da `FeatureAuth.User` va `FeatureProfile.User` turli turlar, ularni bir-biriga uzatib bo'lmaydi.

---

## Task 19 — Umbrella modul `build.gradle.kts`

`settings.gradle.kts`'da:

```kotlin
include(":umbrella")
include(":feature:auth")
include(":feature:profile")
```

**umbrella/build.gradle.kts**

```kotlin
plugins {
    kotlin("multiplatform")
}

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.binaries.framework {
            baseName = "Shared"
            isStatic = true
            export(project(":feature:auth"))
            export(project(":feature:profile"))
        }
    }

    sourceSets {
        commonMain.dependencies {
            api(project(":feature:auth"))
            api(project(":feature:profile"))
        }
    }
}
```

`export` ishlashi uchun bog'liqlik **`api`** bilan e'lon qilingan bo'lishi shart.

---

## Task 20 — Swift `:feature:auth` turlarini ko'rmaydi

**Sabab:** ikkita narsa yetishmayapti: (1) bog'liqlik `implementation` bilan e'lon qilingan, ya'ni tashqariga ochilmaydi; (2) framework blokida `export(...)` yo'q. Kotlin/Native framework'ning header'iga faqat modulning o'z e'lonlari va **aniq `export` qilingan** bog'liqliklar kiradi.

**To'g'rilangan kod:**

```kotlin
kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach {
        it.binaries.framework {
            baseName = "Shared"
            isStatic = true
            export(project(":feature:auth"))   // YANGI
        }
    }
    sourceSets.commonMain.dependencies {
        api(project(":feature:auth"))          // implementation → api
    }
}
```

Faqat `export` yozilib, `api` yozilmasa, Gradle "exported dependencies are not specified as API-dependencies" xatosini beradi.

---

## Task 21 — Nega Kotlin bog'liqligini ikki alohida framework o'rtasida ulashib bo'lmaydi

- Har bir iOS framework **mustaqil, to'liq linklangan binary**. Kotlin/Native kompilyatori uning kodini va barcha bog'liqliklarini (klib'lar) birga bitta binary ichiga yig'adi, shu jumladan Kotlin runtime'ni ham.
- Frameworklar bir-biridan **bexabar** va alohida yig'iladi, birining Kotlin kodiga ikkinchisi havola qila olmaydi. Bir xil kutubxona kodi ikkala binary'da ham alohida nusxa, alohida sinflar va alohida singleton'lar sifatida paydo bo'ladi.
- **Umbrella framework** esa butun bog'liqlik grafini **bitta kompilyatsiya/linklash bosqichida** ko'radi. Shuning uchun har bir kutubxona bitta nusxada, bitta runtime bilan va bitta obyekt nusxalari bilan qo'shiladi. Bitta nusxani kafolatlashning yagona yo'li shu.

---

## Task 22 — Monorepo, 2 repo, 3 repo

| Tuzilma | Afzallik | Kamchilik | Shared qanday versiyalanadi |
|---|---|---|---|
| **Monorepo** | Bitta PR bilan Android, iOS va shared'ni atomik o'zgartirish; publish kerak emas. | iOS dasturchilar ham Kotlin/Gradle toolchain'ga muhtoj; repo va CI og'irlashadi. | Alohida versiyalanmaydi, repo commit'i/ilova relizi bilan birga. |
| **2 repo** (Android+shared \| iOS) | Android va shared birga o'zgaradi; iOS tayyor binary oladi, Kotlin kerak emas. | iOS faqat e'lon qilingan versiyani oladi; o'zgarish kechikib yetadi. | Android manba sifatida ishlatadi (versiyasiz); **iOS uchun binary** (XCFramework/SPM) versiyalanadi. |
| **3 repo** (Android \| iOS \| shared) | Jamoalar to'liq mustaqil, shared qayta ishlatiladi, o'z release sikli bor. | Eng katta overhead: har o'zgarish publish + ikkala ilovada versiya yangilash. | Shared **mustaqil semver** bilan versiyalanadi va ikkala platforma uchun e'lon qilinadi. |

---

## Task 23 — 3 repo: Android/JVM uchun publish qilish

Ikki variant (ma'ruza formulirovkasiga qarab nomlarini moslashtiring):

1. **Bitta KMP Maven publikatsiyasi** (`maven-publish`): bitta koordinata (`group:artifact:version`) ostida Android/JVM va iOS (klib) artefaktlari Gradle metadata bilan birga e'lon qilinadi. Gradle iste'molchi kerakli variantni o'zi tanlaydi.
2. **Platformalar bo'yicha alohida kanallar:** Android/JVM uchun Maven artefakt (AAR/JAR), iOS uchun alohida **XCFramework** (SPM/CocoaPods).

**Trade-off:**
- 1-variant oddiy: bitta versiya, bitta pipeline, izchillik. Lekin Xcode'dagi sof Swift loyiha klib'ni to'g'ridan-to'g'ri ishlata olmaydi, iOS uchun baribir XCFramework kerak bo'ladi.
- 2-variant har bir iste'molchiga o'ziga qulay formatni beradi (Swift'chilar uchun SPM), lekin ikkita release pipeline va ularning versiyalarini sinxron saqlash kerak, ya'ni ko'proq mehnat va xato ehtimoli.

---

## Task 24 — `maven-publish` va mavenLocal

**shared/build.gradle.kts**

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
    id("maven-publish")
}

group = "uz.ttpu.shared"
version = "1.0.0"

kotlin {
    androidTarget {
        publishLibraryVariants("release")
    }
    jvm()
    iosArm64()
    iosSimulatorArm64()
}

android {
    namespace = "uz.ttpu.shared"
    compileSdk = 35
    defaultConfig { minSdk = 24 }
}

publishing {
    repositories {
        mavenLocal()
    }
}
```

Publish qilish:

```bash
./gradlew :shared:publishToMavenLocal
```

**Android repo'sida iste'mol qilish.** `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        mavenLocal()     // YANGI
        google()
        mavenCentral()
    }
}
```

`app/build.gradle.kts`:

```kotlin
dependencies {
    implementation("uz.ttpu.shared:shared:1.0.0")
}
```

---

## Task 25 — XCFramework va Swift package

**shared/build.gradle.kts**

```kotlin
import org.jetbrains.kotlin.gradle.plugin.mpp.apple.XCFramework

plugins {
    kotlin("multiplatform")
}

kotlin {
    val xcf = XCFramework("Shared")

    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.binaries.framework {
            baseName = "Shared"
            isStatic = true
            xcf.add(this)
        }
    }
}
```

**Yig'ish va checksum:**

```bash
./gradlew :shared:assembleSharedReleaseXCFramework
# natija: shared/build/XCFrameworks/release/Shared.xcframework

cd shared/build/XCFrameworks/release
zip -r Shared.xcframework.zip Shared.xcframework
swift package compute-checksum Shared.xcframework.zip
```

**Package.swift** (iOS repo'da yoki alohida paket repo'sida)

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "Shared",
    platforms: [.iOS(.v15)],
    products: [
        .library(name: "Shared", targets: ["Shared"])
    ],
    targets: [
        .binaryTarget(
            name: "Shared",
            url: "https://example.com/releases/1.0.0/Shared.xcframework.zip",
            checksum: "<swift package compute-checksum natijasi>"
        )
    ]
)
```

**Nega checksum shart:** binary targetda ko'rib chiqish uchun manba kod yo'q. SwiftPM yuklab olingan zip **aynan e'lon qilingan fayl ekanligini** checksum orqali tekshiradi. Bu faylning buzilishi, almashtirilishi yoki yo'lda o'zgartirilishidan (tampering, MITM) himoya qiladi, shuningdek build'ni qayta ishlab chiqariladigan qiladi: bir xil URL ostidagi fayl o'zgarsa, checksum mos kelmaydi va SwiftPM build'ni rad etadi.

---

## Task 26 — Local (source) vs Remote (artifact)

| Mezon | Local (manba) | Remote (artefakt) |
|---|---|---|
| **iOS dasturchi onboarding** | Kotlin/JDK/Gradle toolchain o'rnatish, birinchi build uzoq. | Faqat SPM paketini qo'shadi, Kotlin kerak emas. |
| **Iteratsiya tezligi** | Tez: shared'ni o'zgartirib, darhol Xcode'da ko'radi. | Sekin: o'zgartirish → publish → versiya yangilash → build. |
| **Versiyalash overhead'i** | Yo'q (bir xil manba/commit). | Bor: semver, release pipeline, changelog, mosligi. |
| **Shared kodni kim tahrirlay oladi** | Hamma, shu jumladan iOS dasturchilar. | Odatda faqat shared repo'ga kirish va release huquqi borlar; iOS faqat iste'molchi. |

---

## Task 27 — Hybrid workflow (iOS dasturchi shared kodni o'zgartirishi kerak)

1. **Standart holat:** iOS ilova shared'ni **remote** binary paket (URL + checksum, versiya X) orqali oladi.
2. **Lokalga o'tish:** shared repo'ni ilova yoniga clone qiling. Lokal XCFramework yig'ing:
   `./gradlew :shared:assembleSharedReleaseXCFramework`
   So'ng `Package.swift`'da remote `binaryTarget`ni lokal yo'lga almashtiring:
   `.binaryTarget(name: "Shared", path: "../shared/build/XCFrameworks/release/Shared.xcframework")`
   (yoki Xcode'da "Add Local Package").
3. **Tekshirish:** ilovani build/run qiling, shared testlarini (`iosSimulatorArm64Test`) va ilova testlarini ishga tushiring, o'zgarishni qo'lda sinang.
4. **Publish:** shared repo'da PR oching, review'dan o'tkazing va merge qiling. CI yangi XCFramework yig'adi, tag qo'yadi (masalan, `1.1.0`), zip'ni yuklaydi va checksum'ni e'lon qiladi.
5. **Remote'ga qaytish:** lokal yo'lni olib tashlab, remote URL'ga qaytaring, **yangi versiya va checksum** bilan, va shuni commit qiling.

**To'g'ri tartib:** **avval publish, keyin ilovani yangilash.** Ilova hali mavjud bo'lmagan versiyaga havola qilsa, CI va boshqa dasturchilarning build'i buziladi.

---

## Task 28 — Repository grafi (App A va App B)

```
 ┌───────────┐      ┌─────────────┐      ┌───────────┐
 │  logging  │      │ networking  │      │ analytics │      ← kutubxona repo'lari
 └─────┬─────┘      └──────┬──────┘      └─────┬─────┘
       │                ┌──┴───┐                │
       │                │      │                │
       ▼                ▼      ▼                ▼
   ┌──────────────────────┐  ┌──────────────────────┐
   │  app-a-umbrella      │  │  app-b-umbrella      │      ← har ilova uchun umbrella repo
   │  (logging+networking)│  │ (networking+analytics)│
   └──────────┬───────────┘  └──────────┬───────────┘
              ▼                         ▼
        App A (Android, iOS)      App B (Android, iOS)
```

**Nega har bir ilovaga o'z umbrella repo'si:**

- Har bir ilova **faqat o'ziga kerakli** kutubxonalarni oladi (A: logging + networking, B: networking + analytics). Umumiy umbrella A'ga keraksiz `analytics`ni, B'ga keraksiz `logging`ni qo'shib yuborardi.
- iOS'da ilova uchun **bitta framework** bo'lishi kerak (har bir kutubxona bitta nusxada), shuning uchun har bir ilovaning umbrella'si o'z to'plamini yig'adi.
- Ilovalar relizlari va kutubxona versiyalari bir-biridan mustaqil bo'ladi.

---

## Task 29 — Konfiguratsiya tanlash

| Holat | Modullar | Repo'lar | Distribution | Sabab |
|---|---|---|---|---|
| **Yakka dasturchi, kichik ilova** | Bitta `:shared` | Monorepo | Local (manba) | Eng kam murakkablik, publish va versiyalash kerak emas. |
| **4 muhandis, Android+iOS birga reliz** | Bitta yoki bir nechta feature modul + iOS uchun umbrella | Monorepo | Local (manba) | Atomik o'zgarishlar, relizlar bir vaqtda, kichik jamoa hamma toolchain'ni ushlab turadi. |
| **Alohida Android va iOS jamoalari, mustaqil relizlar** | Bir nechta modul + umbrella | 2 repo (Android+shared \| iOS) | Remote (XCFramework), kerak bo'lsa hybrid | iOS jamoasi Kotlin toolchain'siz mustaqil ishlaydi va o'z tezligida reliz qiladi; shared versiyalanadi. |

---

## Task 30 — 40k qatorli `:shared`ni iOS API'sini o'zgartirmasdan bo'lish

1. **Boshlang'ich holatni qayd eting:** build vaqtini o'lchang (Gradle build scan). Hozirgi iOS API'ni saqlang: framework `baseName = "Shared"` va generatsiya qilingan `Shared.h` header'ining nusxasi. Kotlin'da `apiDump` (binary-compatibility-validator) ham mumkin.
2. **Umbrella modulni yarating** (`:umbrella`). Framework konfiguratsiyasini (`baseName = "Shared"`, `isStatic = true`) shu yerga ko'chiring. Eski `:shared` oddiy kutubxonaga aylanadi. iOS ilova hali ham `Shared`ni import qiladi, ya'ni hech narsa o'zgarmaydi.
3. **Eng quyi qatlamni ajrating:** `:core` (modellar, yordamchi funksiyalar). **Package nomlarini o'zgartirmang**, kodni ko'chiring, `api(project(":core"))` qo'shing.
4. **Feature modullarni birma-bir ajrating** (`:feature:auth`, `:feature:profile`, ...). Feature'lar bir-biriga to'g'ridan-to'g'ri bog'lanmasin, umumiy narsa `:core` orqali o'tsin (tsikllardan saqlaning). Public API'da ko'rinadigan turlar uchun `api` ishlating.
5. **Umbrella'da hammasini ulang:** har bir modulni `api(project(...))` bilan qo'shing va framework blokida `export(project(...))` qiling.
6. **Har bir ajratishdan keyin tekshiring:** build, testlar, `apiDump` o'zgarmaganligi, yangi `Shared.h` eski bilan solishtiriladi (Swift'da ko'rinadigan nomlar bir xil bo'lishi kerak) va iOS ilova **kodni o'zgartirmasdan** kompilyatsiya bo'ladi.
7. **Tezlashtirish:** `gradle.properties`'da `org.gradle.parallel=true`, `org.gradle.caching=true`, configuration cache; build vaqtini 1-bosqichdagi bilan solishtiring.
8. **Tozalash:** eski `:shared`'dan ko'chirilgan kodni olib tashlang, bo'sh qolsa modulni o'chiring yoki umbrella sifatida qoldiring.
