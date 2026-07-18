# MotorNet Android Localization and QA Workflow

This document describes an independent, reproducible workflow for localizing
and validating the MotorNet Android user interface. It is intended for
contributors working with software they are authorized to modify.

It does not contain an APK, application assets, extracted proprietary strings,
signing keys, controller firmware, or FarDriver source code. It is not an
official FarDriver release or signing process.

## Scope

The workflow covers:

* terminology preparation and translation review;
* placeholder and technical-value preservation;
* authorized Android build or repack validation;
* installation and launch checks with ADB;
* screen-level localization coverage;
* crash collection with `logcat`;
* a concise QA report suitable for review by a vendor or maintainer.

The preferred input is a vendor-provided localization export or source build.
An internally modified APK should be treated only as a feasibility prototype
until the rights holder approves the work and produces an officially signed
release.

## Translation Invariants

A translation pass must preserve machine-readable token identity and syntax:

* .NET indexed format items such as `{0}`, `{1}`, and `{0:N2}`;
* named placeholders such as `{controller}`;
* printf-style placeholders such as `%s`, `%d`, `%.1f`, and `%2$s`;
* escaped sequences such as `\n`, `\t`, `\\`, and XML entities;
* escaped format literals such as `{{`, `}}`, and `%%`;
* enum members, protocol identifiers, model names, and CAN identifiers;
* hexadecimal values, checksums, addresses, and firmware filenames;
* units such as `V`, `A`, `W`, `rpm`, `°C`, `ms`, and `%`;
* Bluetooth device names and serial command payloads.

Do not translate short identifiers merely because they look like English. For
example, `CAN`, `RXD`, `TXD`, `SOC`, `BOOST`, and protocol field names may be
part of a hardware or wire-level contract.

Placeholder validation must follow the formatter's semantics:

* for .NET indexed and named placeholders, preserve identity, multiplicity, and
  format specifiers; target-language word order may reorder the placeholders;
* for non-positional printf placeholders, preserve sequence and type;
* reorder indexed printf placeholders only when the target formatter supports
  that syntax;
* give translators context describing each argument and an example value.

Automated checks should parse each placeholder family instead of comparing one
raw ordered list. Reject changed token identities or counts, malformed tokens,
changed format specifiers, and illegal reordering of non-positional printf
placeholders. This distinction follows Microsoft's
[placeholder localization guidance][placeholder-guidance] and the
[.NET composite formatting contract][composite-formatting].

## Xamarin and AssemblyStore Note

MotorNet builds encountered in the field may use Xamarin/Mono AssemblyStore.
In that layout, adding Android resources under `res/values-ru` can translate
native Android and library strings while leaving managed XAML or .NET strings
unchanged. A successful launch therefore does not prove that the application
is localized.

For an official release, managed strings should be changed through the vendor's
source tree or localization export. When an authorized internal feasibility
build rewrites AssemblyStore content, it must rebuild the matching store
metadata and preserve package compression and alignment requirements. Direct
byte replacement that changes an assembly's length without rebuilding the
metadata is not valid. The final package still needs the vendor's release
signing process.

## Workflow

### 1. Establish the baseline

Record the following before changing anything:

* application version and package identifier;
* APK SHA-256 checksum and signer certificate fingerprint;
* Android version, device model, and display size;
* application language at first launch;
* a list of reachable screens and dialogs;
* known crashes, permission failures, and connectivity requirements.

Keep the original package immutable. Never place signing keys or credentials in
the repository or QA report.

### 2. Build a terminology glossary

Review terminology against the public controller documentation in this
repository, especially [MANUAL.md](MANUAL.md), [README.md](README.md), and the
field names in [fardriver.hpp](fardriver.hpp). Use one translation for one
technical concept, and flag ambiguous source terms instead of guessing.

Safety-related settings need human technical review. A fluent translation that
confuses phase current with battery current is worse than an untranslated
label.

### 3. Translate and validate

For each string:

1. classify it as user-facing text, a technical identifier, or uncertain;
2. translate only user-facing text;
3. preserve whitespace and placeholders required by the UI framework;
4. run placeholder, unit, and identifier checks;
5. review text length for compact controls and narrow displays;
6. record intentionally untranslated terms in an allowlist.

Machine translation can produce a draft, but controller terminology and every
write/reset/firmware action require human review.

### 4. Produce an authorized test build

Use the vendor build whenever it is available. For an internal feasibility
prototype, use a separate debug signing identity and do not publish the APK.
Record the resulting SHA-256 checksum and signer fingerprint in the private QA
record.

A debug-signed APK may not update an officially signed installation. Uninstalling
the official application can remove its local data, so back up permitted data
and use a dedicated test device.

### 5. Install and launch with ADB

Example commands, with the APK path adjusted locally:

```sh
adb devices -l
adb install -r path/to/authorized-test.apk
adb shell am force-stop com.FarDriver.MotorNet
adb logcat -c
adb shell monkey -p com.FarDriver.MotorNet \
  -c android.intent.category.LAUNCHER -v 1
```

The package must launch more than once, including after a force-stop and after a
device reboot. The Monkey command is a launch smoke test; its verbose output
must identify the expected package and activity. It is not deterministic UI
automation and does not prove screen coverage or controller connectivity. See
the [Android Monkey documentation][android-monkey] for option semantics.

### 6. Walk the UI

Exercise every reachable screen in a deterministic order:

1. first launch, permissions, and privacy prompts;
2. account or offline entry path;
3. Bluetooth scan and connection states;
4. controller status and read-only parameter views;
5. settings categories, menus, dialogs, validation errors, and confirmations;
6. disconnected, timeout, malformed-data, and retry states;
7. rotation, font scaling, dark mode, and the smallest supported display.

Evidence may be a screen checklist, screen recording, or before/after images,
provided it contains no credentials, personal data, or device identifiers that
should remain private.

### 7. Collect crashes

After each test pass, collect the application, system, and crash buffers for the
documented test window:

```sh
adb logcat -d -b main -b system -b crash -v threadtime \
  > motornet-logcat.txt
adb shell dumpsys package com.FarDriver.MotorNet \
  > motornet-package.txt
```

Logcat buffers are device-wide. Attribute a failure by test time, package,
process, and stack trace; a crash from another package is not a MotorNet failure.
When investigating MotorNet, inspect `AndroidRuntime`, `ActivityManager`, `Mono`,
`Xamarin`, and native linker messages. A clean crash buffer alone does not rule
out an ANR. Redact tokens, account data, Bluetooth addresses, and serial numbers
before sharing logs. See the [Android logcat documentation][android-logcat] for
buffer semantics.

Treat the following as release blockers:

* `FATAL EXCEPTION`, native aborts, ANRs, or repeated process restarts;
* missing managed assemblies or AssemblyStore lookup failures;
* resource-not-found, format, or placeholder exceptions;
* a screen that cannot be exited without force-stopping the application;
* controller values or units changed by localization.

## Controller Safety Gate

Localization QA should be read-only by default. Do not combine a language test
with controller firmware updates or unreviewed parameter writes.

When hardware interaction is necessary:

* use a controlled bench setup and prevent the drive wheel from contacting the
  ground;
* save the original parameter set before any authorized write test;
* verify voltage, current, temperature, direction, and speed units independently;
* require an explicit test case for Save, Reset, Self-learning, and Firmware
  Update actions;
* stop when the displayed value disagrees with the controller documentation or
  a known-good reference application.

## Acceptance Criteria

A localization candidate is ready for vendor review only when:

* the APK installs and launches repeatedly on the target device matrix;
* no MotorNet-related fatal exception, native abort, ANR, or repeated process
  restart occurs during the documented test window;
* all planned screens are covered, not only Android resource dialogs;
* placeholder identities, counts, and format specifiers match the source;
* identifiers, values, and units match the source;
* no clipped or overlapping text blocks the workflow;
* remaining source-language text is classified and documented;
* controller readouts match a known-good baseline;
* no modified APK or signing material is publicly distributed.

Coverage should be reported as both string coverage and screen coverage. String
counts alone can be misleading when strings are unused, duplicated, or stored
inside managed assemblies.

## QA Report Template

```text
Application version:
Source APK SHA-256:
Test APK SHA-256:
Signing mode: vendor / internal debug
Device and Android version:
Locale:
Test window (start/end):

Strings inventoried:
Strings translated:
Strings intentionally unchanged:
Placeholder validation:

Screens planned / passed / failed:
Cold app launches passed:
Device reboot launch passed:
Controller connection tested: no / read-only / authorized write
MotorNet crash / ANR result:

Known issues:
Artifacts retained privately:
Reviewer:
Date:
```

## Starter English-Russian Glossary

These are generic controller terms derived from the public documentation in
this repository. Product-specific labels still need context review.

| English | Russian | Note |
| --- | --- | --- |
| Controller | Контроллер | Motor controller |
| Motor | Двигатель | |
| Rated voltage | Номинальное напряжение | Preserve `V` |
| Rated power | Номинальная мощность | Preserve `W`/`kW` |
| Rated speed | Номинальная частота вращения | Preserve `rpm` |
| Maximum phase current | Максимальный фазный ток | Not battery current |
| Maximum line current | Максимальный ток батареи | Battery-bus current |
| Pole pairs | Пары полюсов | |
| Hall sensor | Датчик Холла | |
| Position sensor | Датчик положения | |
| Throttle | Ручка газа / педаль акселератора | Vehicle-dependent |
| Low throttle threshold | Нижний порог сигнала газа | Preserve voltage |
| High throttle threshold | Верхний порог сигнала газа | Preserve voltage |
| Self-learning | Самообучение | Safety-sensitive action |
| Motor direction | Направление вращения | |
| Reverse speed | Скорость заднего хода | |
| Speed limit | Ограничение скорости | |
| Field weakening | Ослабление поля | Avoid literal ambiguous wording |
| Regenerative braking | Рекуперативное торможение | |
| Brake current | Ток торможения | Preserve `A` |
| BOOST | BOOST (форсированный режим) | Preserve protocol/display token |
| Cruise control | Круиз-контроль | |
| Overvoltage protection | Защита от перенапряжения | |
| Undervoltage protection | Защита от пониженного напряжения | |
| Motor temperature | Температура двигателя | Preserve `°C` |
| Controller temperature | Температура контроллера | Preserve `°C` |
| Read controller data | Считать параметры контроллера | |
| Write controller data | Записать параметры контроллера | Confirm first |
| Firmware update | Обновление прошивки | Safety-sensitive action |
| Factory reset | Сброс к заводским настройкам | Confirmation required |
| Error code | Код ошибки | Preserve numeric/hex value |
| Serial number | Серийный номер | Do not expose in public logs |
| Connect / Disconnect | Подключить / Отключить | |
| Save / Cancel | Сохранить / Отмена | |

## Contributing Results

Public contributions should contain methodology, glossary corrections, and
redacted reports only. Keep APKs, extracted resources, private controller dumps,
credentials, certificates, and signing material out of Git history.

When proposing a glossary change, link it to the relevant section of the public
manual or protocol definition and explain the engineering meaning, not only the
literal translation.

## References

* [Microsoft localization guidance for placeholders][placeholder-guidance]
* [Microsoft .NET composite formatting][composite-formatting]
* [Android UI/Application Exerciser Monkey][android-monkey]
* [Android logcat command-line tool][android-logcat]

[placeholder-guidance]: https://learn.microsoft.com/en-us/globalization/internationalization/contextual-metadata
[composite-formatting]: https://learn.microsoft.com/en-us/dotnet/standard/base-types/composite-formatting
[android-monkey]: https://developer.android.com/studio/test/other-testing-tools/monkey
[android-logcat]: https://developer.android.com/tools/logcat
