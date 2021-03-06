## Качество кода и настройки билда для приложений iOS

### Проверка правильности подписи приложения

#### Обзор

Подпись кода для приложений используется для заверения пользователей, что оно поступило из известного источниа и приложение не было изменено с момента последней подписи. До того момента как приложение может интегрироваться с сервисами(app services), быть установленным на устройство или же отправлено в Apple Store, оно должно быть подписано сертификатом, выданым Apple. Для более детальной информации о получении сертификатов обращайтесь к [руководству App Distribution](https://developer.apple.com/library/content/documentation/IDEs/Conceptual/AppDistributionGuide/Introduction/Introduction.html).

Возможно получить информацию о сертификате, использовав на файле приложения .app [codesign](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/codesign.1.html). Codesign используется для создания, проверки и отображения подписей кода, а также для запроса динамического статуса подписанного кода в системе.

После получения файла .ipa приложения, переименуйте его в формат zip и разархивируйте его. Перейдите в папку `Payload` и увидите файл .app.

Выполните команду `codesign`:

```shell
$ codesign -dvvv <yourapp.app>
Executable=/Users/Documents/<yourname>/Payload/<yourname.app>/<yourname>
Identifier=com.example.example
Format=app bundle with Mach-O universal (armv7 arm64)
CodeDirectory v=20200 size=154808 flags=0x0(none) hashes=4830+5 location=embedded
Hash type=sha256 size=32
CandidateCDHash sha1=455758418a5f6a878bb8fdb709ccfca52c0b5b9e
CandidateCDHash sha256=fd44efd7d03fb03563b90037f92b6ffff3270c46
Hash choices=sha1,sha256
CDHash=fd44efd7d03fb03563b90037f92b6ffff3270c46
Signature size=4678
Authority=iPhone Distribution: Example Ltd
Authority=Apple Worldwide Developer Relations Certification Authority
Authority=Apple Root CA
Signed Time=4 Aug 2017, 12:42:52
Info.plist entries=66
TeamIdentifier=8LAMR92KJ8
Sealed Resources version=2 rules=12 files=1410
Internal requirements count=1 size=176
```

### Поиск отладочных символов

#### Обзор

Как общее правило, чем меньше объяснительной информации существует в скомпилированом коде, тем лучше. Такие метаданные как отладочная информация, номера строчек кода и описательные имена функций или методов делают бинарный или байт код намного легче для понимания инженеров обратного проектирования, но на самом деле все это не требуется в релиз сборке и следовательно может быть с легкостью убрано, без влияния на функциональность приложения.

Такие символы могут быть сохранены либо в формате Stabs либо в DWARF. При использовании формата Stubs, отладочные символы, как и другие символы, хранятся в регулярной таблице символов. С форматом DWARF, отладочные символы хранятся в специальном сегменте "\_\_DWARF" в бинарнике. Отладочные символы DWARF могут быть сохранены как отдельный файл с отладочной информацией. В данном случае, вы проверяете, что никаких отладочных символов нет в самом релизном бинарном файле(или же в таблице символов или в сегменте \_\_DWARF).

#### Статический анализ

Используйте gobjdump для изучения основного бинарного файла и любых включенных динамических библиотек(dylibs) на предмет наличия символов Stabs или DWARF.

```shell
$ gobjdump --stabs --dwarf TargetApp
In archive MyTargetApp:

armv5te:     file format mach-o-arm


aarch64:     file format mach-o-arm64
```

Gobjdump - часть [binutils](https://www.gnu.org/s/binutils/ "Binutils") и может быть установлена через Homebrew на Mac OS X.

#### Динамический анализ

Динамический анализ не применим для поиска отладочных символов.

#### Исправление

Обедитесь, что отладочные символы вырезаны, во время билда в продакшен. Их удаление снизит размер бинарного файла и увеличит сложность обратного проектирования. Для того чтобы вырезать отладочные символы, выставите `Strip Debug Symbols During Copy` в значение YES в настройках билдка(build settings) в вашем проекте.

Также возможно иметь хорошую [систему Crash Reporter](https://developer.apple.com/library/content/documentation/IDEs/Conceptual/AppDistributionGuide/AnalyzingCrashReports/AnalyzingCrashReports.html), так как она не требует никаких символов в бинарнике приложения.

### Нахождение отладочного кода и подробного лога ошибок

#### Обзор

Разработчики частно включают отладочный код, такой как подробное логгирование(используя  `NSLog`, `println`, `print`, `dump`, `debugPrint`), для информирования о ответах с собственного API, о прогрессе и/или состоянии своего приложения для того чтобы ускорить процесс нахождения и исправления ошибок. Более того, может существовать отладочный код, воспринимаемый как "функциональность менеджмента", которая используется разработчиками, чтобы установить необходимое состояние приложению, эмулировать(mock) ответы от API и тому подобное. Данная информация с легкостью может использоваться инженерами обратного проектирования для отслеживания того, что происходит с приложением. Поэтому отладочный код должен быть удален из релизных сборок приложений

#### Статический анализ

Для статического анализа вы можете воспользоваться следующим подходом, в зависимости от функции логирования:

 1. Импортировать код приложения в Xcode.
 2. Сделайте поиск в проекте следующих функций печати:`NSLog`, `println`, `print`, `dump`, `debugPrint`.
 3. Когда одна из них найдена, проверьте используют ли разработчики функцию обертки для более удобного форматирования вывода, если да, то включите и их в поиск.
 4. Для каждого совпадения, найденного на шаге 2 и 3, проверьте используется ли макрос или защита отладочного состояния(debug-state related guards) для выключения логирования в релиз-сборке. Обратите внимание на изменение в том как Objective-С может использовать препроцессорный макрос:

```objc
#ifdef DEBUG
    // Debug-only code
#endif
```

В Swift процедура включения такого поведения изменилась: здесь вам необходимо установить переменную среды в вашей схеме или установить нестандартный флаг в настройках билда цели(target), чтобы это заработало. Обратите внимание, что использовать следующие функции, которые позволяют определить собиралось ли приложении с настройками релиз в Swift 2.1 не рекомендуется(так как Xcode 8 и Swift3 не поддерживает их):
<!---TODO(зачеркнуто, оставил специально) что за **** в ориганиле:Please note that the following functions, which allow to check on whether the app is build in release-configuration in Swift 2.1, should be recommended against (As Xcode 8 & Swift3 do not support them): ...   --->  

- `_isDebugAssertConfiguration()`
- `_isReleaseAssertConfiguration()`
- `_isFastAssertConfiguration()`

Обратите внимание, что существует много функций логгирования, в зависимости от настройки приложения, например, когда используется [CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack "CocoaLumberjack"), то статический анализ немного отличается.

Для кода "отладочного менеджмента(управления)", который встроен: исследуйте файлы .storyboards на предмет обнаружения каких-либо потоков управления viewControllers, которые предоставляют отличающуюся функциональность от ожидаемой, поддерживаемой приложением. Это может быть что угодно: от отладочных view до отрисовки сообщений о ошибке; от кастомных эмулирований ответов с сервера до записи логов в локальный или удаленные на сервере файлы.

#### Динамический анализ

Динамический анализ должен быть осуществлен и на симуляторе и на реальном устройстве, так как иногда случается что разработчики используют функции, основанные на информации о устройстве, нежели основанные на установленном режиме: отладки или релиза, для выполнения отладочного кода.

1. Запустите приложение на симуляторе, проверте есть ли какой-либо вывод в консоли, во время использования приложения.
2. Подключите устройство к маку, запустите приложение на устройстве, используя Xcode, и проверте есть ли что-то в выводе консоли.

Для другого кода, "основанного на управлении(состояниями приложения и т.д.)": прокликайте приложение на устройстве и симуляторе и посмотрите сможете ли вы найти функции, которые позволяют воспользоваться защитыми профилями в приложении, выбрать сервер, выбрать возможные ответы от API и тому подобное.

#### Исправление

Как разработчик, вы не будете испытывать проблем с добавлением отладочных функций в вашу отладочную версию приложения ровно до тех пор, пока вы помните, что отладочный код никогда не должен:

- оставаться в релизной версии приложения
- оставаться в релизной конфигурации приложения

В Objective-C, разработчики могут использовать препроцессорные макросы для фильтрации отладочной печати:

```objc
#ifdef DEBUG
    // Debug-only code
#endif
```

В Swift 2, при использовании  Xcode 7, необходимо выставить нестандартный флаг компилятора для каждой цели(target) сборки, где флаг компилятора должен начинаться с -D. Таким образом, если флаг отладки -DMSTG-DEBUG установлен, можете использовать следующую аннотацию:

```swift
#if MSTG-DEBUG
    // Debug-only code
#endif
```

В Swift 3, при использовании  Xcode 8, вы можете выставить Active Compilation Conditions в Build settings/Swift compiler - Custom flags. Swift3 не использует препроцессор, но вместо этого использует [условные блоки компиляции](https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html#//apple_ref/doc/uid/TP40014216-CH8-ID34 "Swift conditional compilation blocks"), основанные на предопределенных условиях:

```swift
#if DEBUG_LOGGING
    // Debug-only code
#endif
```

### Проверка обработки исключений

#### Обзор

Исключения часто могут возникать, когда приложение попадает в ненормальное или ошибочное состояние. Проверка обработки исключений заключается в перепроверке того, что при возникновении исключений приложение сможет попасть в безопасное состояние без раскрытия чувствительной информации в пользовательском интерфейсе или же механизме логирования приложения.

Однако, имейте в виду, что обработка ошибок в Objective-C отличается от Swift. Совмещение обоих языков и концепций в проекте может быть проблематично, с точки зрения обработки ошибок.

##### Обработка исключений в Objective-C

В Objective-C существует два типа ошибок:

###### NSException

`NSException` используется для обработки программных или низкоуровневых ошибок(например, деление на 0, выход за границы массива).
`NSException` можно вызвать функцией `raise()` или "бросить", используя `@throw`, и в случае, если ошибка не была поймана, то произойдет вызов обработчика необработанных исключений, где вы можете вывести в лог информацию, после чего программа будет остановлена, `@catch` позволяет избежать остановки, если вы используете блок `@try`-`@catch`:

```objc
 @try {
    //do work here
 }

@catch (NSException *e) {
    //recover from exception
}

@finally {
    //cleanup
```

Помните, что использование NSException таит в себе подводные камни, относительно управления памятью, вам нужно [очищать аллоцированную память](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Exceptions/Tasks/RaisingExceptions.html#//apple_ref/doc/uid/20000058-BBCCFIBF "Raising exceptions")  из блока try в [finally block](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Exceptions/Tasks/HandlingExceptions.html "Handling Exceptions"). Обратите внимание, что вы можете превратить объекты `NSException` в `NSError`, если объявите `NSError` в блоке `@catch`.

###### NSError

`NSError` используется для всех остальных типов [ошибок](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/ErrorHandling/ErrorHandling.html "Dealing with Errors"). Некоторые API библиотеки Cocoa предоставляют эти объекты в обратный вызов сбоя, если что-то пошло не так, в противном случае предоставляется ссылка на объект `NSError`. Хорошая практика - объявить возвращаемый тип функции как булевый, у функции, которая принимает как аргумент ссылку на объект `NSError` и  оригинально ничего не должна возвращать, для того чтобы сигнализировать о успехе или неудачи операции. Если же у функции имеется возвращаемое значение, то необходимо возвращать nil, в случае ошибки. Таким образом, в случае возврата NO или nil, вы можете разбираться в причинах ошибки или сбоя.

##### Обработка исключений в Swift

Обработка исключений в Swift (2~4) немного отличается. Даже если имеется блок try-catch, то он находится там не для обработки NSException. Вместо этого, он используется для обработки ошибок, которые поддерживает протокол `Error` (Swift3, `ErrorType` в Swift2). При комбинировании кода Objective-C и Swift могут возникнуть сложности. Более того, использование `NSError` рекомендуется вместо `NSException` в программах, где используются оба языка. В дополнение к этому, в Objective-C обработка исключений является добровольной, но в Swift вам необходимо явно обрабатывать `throws`. Для преобразования ошибок посмотрите [документацию Apple](https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/AdoptingCocoaDesignPatterns.html "Adopting Cocoa Design Patterns").
Методы, которые могут выдавать ошибки используют ключевое слово `throws`. Существует четыре способа для [обработки ошибок в Swift](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/ErrorHandling.html "Error Handling in Swift"):  

- Вы можете передавать ошибку из функции в код, который ее вызвал: в данном случае нет do-catch, а есть только `throw`, выдающий действительную ошибку или же есть `try` для исполнения метода, который выкидывает ошибку. Метод, содержащий `try` также должен использовать ключевое слово `throws`:

    ```swift
    func dosomething(argumentx:TypeX) throws {
        try functionThatThrows(argumentx: argumentx)
    }
    ```

- Обработать ошибку, используя do-catch:

    ```swift
    do {
        try functionThatThrows()
        defer {
            //use this as your finally block as with Objective-c
        }
        statements
    } catch pattern 1 {
        statements
    } catch pattern 2 where condition {
        statements
    }
    ```

- Обработать ошибку, как опциональное значение:

    ```swift
        let x = try? functionThatThrows()
        //In this case the value of x is nil in case of an error.
    ```  

- Декларировать, что ошибка не возникнет: используя выражение `try!`.

#### Статический анализ

Посмотрите исходный код и поймете/идентифицируйте как приложение обрабатывает различные ошибки(взаимодействия IPC, вызов функций удаленных сервисов и тому подобное). Здесь приведены некоторые примеры проверок(с разделением по языкам), которые вы можете произвести на данном этапе.

##### Статический анализ в Objective-C

Убедитесь что:

- Приложение использует хорошо спроектированную и унифицированную схему обработки ошибок и исключений.
- Исключения из библиотеки Cocoa обрабатываются корректно.
- Выделенная память в блоке `@try` высвыбождается в блоке `@finally`.
- Для каждого `@throw`, вызывающий метод имеет правильный `@catch` либо на уровне вызываюшего метода либо на уровне объектов `NSApplication`/`UIApplication` для того, чтобы подчистить любую чувствительную информацию и возможно попробовать восстановится из этого состояния.
- Приложение не показывает чувствительную информацию во время обработки исключений у себя в интерфейсе пользователя или в логах, но при этом сообщения о ошибках достаточно подробные, чтобы объяснить пользователю что случилось.
- Любая конфиденциальная информация(ввода или данных аутентификации) всегда стирается в блоке `@finally` в случае с приложением, которые имеет повышенные риски.
- `raise()` используется очень редко, когда прекращение выполнения программы, без дополнительных предупреждений необходимо.
- Объекты `NSError` не содержат информации, которая может привести к утечке чуствительной информации.

##### Статический анализ в Swift

Убедитесь, что:

- Приложение использует хорошо спроектированную и унифицированную схему обработки ошибок и исключений
- Приложение не показывает чуствительную информацию во время обработки исключений у себя в интерфейсе пользователя или в логах, но при этом сообщения о ошибках достаточно подробные, чтобы объяснить пользователю что случилось.
- Любая конфиденциальная информация(ввода или данных аутентификации) всегда-всегда стирается в блоке `defer` в случае с приложением, которое имеет повышенные риски.
- `try!` используется только с надлежащей безопасностью, таким образом, что в действительности, программно проверено, что метод не выкидывает ошибок, если вызван с `try!`.

#### Динамическое тестирование

Существуют различные методы динамического тестирования:

- Введите непредвиденные значения в поля ввода графического интерфейса приложения.
- Проверьте нестандартные схемы url, pasteboard и другие средства контроля взаимодействий внутри приложения.
- Подделайте данные сетевого взаимодействия и/или файлов, храняшихся в приложени.
- В случае Objective-C, можете воспользоваться cycript, чтобы подменить методы и передать такие аргументы, которые могут вызвать исключительную ситуацию.

В большинстве случаев приложение не должно крашится, но вместо этого, оно должно:

- Восстановится после ошибки или перейти в то состояние, где оно сможет проинформировать пользователя, что приложение не может продолжить свою работу.
- Если необходимо, проинформировать пользователя о действиях, которые ему нужно предпринять. Но само сообщение не должно содержать никакой чувствительной информации.
- Не предоставлять никакой информации в лог.

#### Исправление

Разработчик может реализовать правильную обработку ошибок несколькими способами:

- Убедитесь, что приложение использует хорошо спроектированную и унифицированную схему обработки ошибок.
- Убедитесь, что все логирование выключены(убрано) или же использует меры безопасности, описанные в главе "Проверка отладочного кода и подробного лога ошибок".
- Для Objective-C, в случае приложения с повышенным риском: создайте свой обработчик исключений, который будет чистить все секреты, которые не должны быть так легко получены. Обработчик может быть установлен через `NSSetUncaughtExceptionHandler`.
- При использовании Swift, убедитесь, что вы используете `try!` только в том случае, если вы точно знаете что в вызываемом методе не может возникнуть ошибок.
- При использовании Swift, убедитесь, что ошибки не передаются слишком далеко через промежуточные методы.

### Убедитесь, что "бесплатные" свойства безопасности включены

#### Обзор

Несмотря на то, что Xcode выставляет все бинарные свойства безопасности по умолчанию, данная мера все равно применима для некоторых старых приложений или для проверки неправильной настройки компиляции. Следующие свойства применимы:

- **ARC**(Automatic Reference Counting) - свойство управления памятью, добавляет вызовы retain и release по необходимости
- **Stack Canary** - помогает предотвращать атаки по переполнению буфера.
- **PIE**(Position Independent Executable) - включает полный ASLR для бинарника.

#### Статический анализ

##### Xcode настройки проекта

- Защита от Stack smashing

Шаги, необходимые для включения защиты от Stack smashing в приложении iOS:

1. В Xcode, выберите свою цель(target) в секции "Targets", затем нажмите на вкладку "Build Settings" чтобы посмотреть настройки.
2. Проверте, что опция "–fstack-protector-all" выбрана в секции "Other C Flags".

- Поддержка PIE

Шаги для сборки приложения iOS как PIE:

1. В Xcode, выберите свою цель(target) в секции "Targets", затем нажмите на вкладку "Build Settings" чтобы посмотреть настройки.
2. Для приложений iOS, выставьте iOS Deployment Target на iOS 4.3 или позднее.
3. Проверьте что "Generate Position-Dependent Code" установлен в свое значение по умолчанию - NO.
4. Проверьте, что  "Don't Create Position Independent Executables" установлен в свое значение по умолчанию - NO.
<!--- ОПЕЧАТКА?????? Verify that Don't "Create Position Independent Executables" is set at its default value of NO --->

- Защита ARC

Шаги для включения защиты ARC:

1. В Xcode, выберите свою цель(target) в секции "Targets", затем нажмите на вкладку "Build Settings" чтобы посмотреть настройки.
2. Убедитесь, что "Objective-C Automatic Reference Counting" установлен в свое значение по умолчанию - YES.

Также читайте [Technical Q&A QA1788 Building a Position Independent Executable]( https://developer.apple.com/library/mac/qa/qa1788/_index.html "Technical Q&A QA1788 Building a Position Independent Executable").

##### С otool

Ниже приведены примеры того как проверить наличие данных свойств. Обратите внимание, что все они включены в примерах:

- PIE:

    ```shell
    $ unzip DamnVulnerableiOSApp.ipa
    $ cd Payload/DamnVulnerableIOSApp.app
    $ otool -hv DamnVulnerableIOSApp
    DamnVulnerableIOSApp (architecture armv7):
    Mach header
    magic cputype cpusubtype caps filetype ncmds sizeofcmds flags
    MH_MAGIC ARM V7 0x00 EXECUTE 38 4292 NOUNDEFS DYLDLINK TWOLEVEL
    WEAK_DEFINES BINDS_TO_WEAK PIE
    DamnVulnerableIOSApp (architecture arm64):
    Mach header
    magic cputype cpusubtype caps filetype ncmds sizeofcmds flags
    MH_MAGIC_64 ARM64 ALL 0x00 EXECUTE 38 4856 NOUNDEFS DYLDLINK TWOLEVEL
    WEAK_DEFINES BINDS_TO_WEAK PIE
    ```

- Stack Canary:

    ```shell
    $ otool -Iv DamnVulnerableIOSApp | grep stack
    0x0046040c 83177 ___stack_chk_fail
    0x0046100c 83521 _sigaltstack
    0x004fc010 83178 ___stack_chk_guard
    0x004fe5c8 83177 ___stack_chk_fail
    0x004fe8c8 83521 _sigaltstack
    0x00000001004b3fd8 83077 ___stack_chk_fail
    0x00000001004b4890 83414 _sigaltstack
    0x0000000100590cf0 83078 ___stack_chk_guard
    0x00000001005937f8 83077 ___stack_chk_fail
    0x0000000100593dc8 83414 _sigaltstack
    ```

- Automatic Reference Counting:

    ```shell
    $ otool -Iv DamnVulnerableIOSApp | grep release
    0x0045b7dc 83156 ___cxa_guard_release
    0x0045fd5c 83414 _objc_autorelease
    0x0045fd6c 83415 _objc_autoreleasePoolPop
    0x0045fd7c 83416 _objc_autoreleasePoolPush
    0x0045fd8c 83417 _objc_autoreleaseReturnValue
    0x0045ff0c 83441 _objc_release
    [SNIP]
    ```

##### с idb

IDB автоматизирует процесс проверки сразу для stack canary и поддержки PIE. Выберите целевой бинарный файл в IDB GUI и нажмите кнопку "Analyze Binary…".

![IDB](Images/Chapters/0x06i/idb.png)

### Ссылки

#### OWASP Mobile Top 10 2016

- M7 - Качество кода на стороне клиента - <https://www.owasp.org/index.php/Mobile_Top_10_2016-M7-Poor_Code_Quality>

#### OWASP MASVS

- V7.1: "Приложение подписано и снабжено действительным сертификатом."
- V7.4: "Код отладки был удален, и приложение не регистрирует подробные ошибки или отладочные сообщения."
- V7.6: "Приложение отлавливает и обрабатывает возможные исключения."
- V7.7: "Логика обработки ошибок в элементах управления безопасностью запрещает доступ по умолчанию."
- V7.9: "Активированы бесплатные функции безопасности, предлагаемые инструментальной цепью(toolchain), такие как минификация байтового кода, защита стека, поддержка PIE и автоматический подсчет ссылок."

#### Инструменты

- idb - <https://github.com/dmayer/idb>
