# Статус патчей для Sultan Kernel + KernelSU + SUSFS

## Проблема

При применении патчей KernelSU-Next + SUSFS к ядру Google Tensor (tensynos) с модификациями Sultan возникли конфликты. Патчи не применились автоматически и создались .rej файлы.

## Причина

Исходный код ядра Sultan отличается от стандартного:
- Другая структура includes в `fs/proc/base.c`
- Кастомная логика для GMS в `kernel/sys.c` (функция `newuname()`)
- Возможно другие модификации

## Решение

Созданы адаптированные патчи в директории `kernel_patches/sultan/` и интегрированы в GitHub Actions workflow.

### Патчи:

#### 1. fix_base.c.patch
Добавляет поддержку SUSFS в `fs/proc/base.c`:
```c
#ifdef CONFIG_KSU_SUSFS_SUS_MAP
#include <linux/susfs_def.h>
#endif
```

#### 2. sys.c_fix.patch
Добавляет подмену uname в `kernel/sys.c`:
```c
#ifdef CONFIG_KSU_SUSFS_SPOOF_UNAME
extern void susfs_spoof_uname(struct new_utsname* tmp);
#endif
```

### Интеграция в CI

Патчи автоматически применяются в GitHub Actions workflow (`.github/workflows/sultan.yml`) при сборке с SUSFS:

```yaml
# Apply Sultan-specific patches for tensynos kernel
echo "Applying Sultan-specific patches..."
if [ -f "../kernel_patches/sultan/fix_base.c.patch" ]; then
  echo "Applying fix_base.c.patch..."
  patch -p1 < ../kernel_patches/sultan/fix_base.c.patch || true
fi
if [ -f "../kernel_patches/sultan/sys.c_fix.patch" ]; then
  echo "Applying sys.c_fix.patch..."
  patch -p1 < ../kernel_patches/sultan/sys.c_fix.patch || true
fi
```

## Отклоненные патчи (.rej файлы)

Файлы в `local/rej/android_kernel_google_tensynos/` показывают, что не применилось:

### KernelSU-Next патчи:
- ✗ `kernel/Makefile.rej` - проверка версии SUSFS при сборке
- ✗ `kernel/ksu.c.rej` - условная компиляция с CONFIG_KSU_SUSFS
- ✗ `kernel/allowlist.c.rej` - интеграция с SUSFS
- ✗ `kernel/kernel_umount.c.rej` - интеграция с SUSFS
- ✗ `kernel/ksud.c.rej` - интеграция с SUSFS
- ✗ `kernel/sucompat.c.rej` - интеграция с SUSFS
- ✗ `kernel/supercalls.c.rej` - интеграция с SUSFS

### Kernel патчи:
- ✓ `fs/proc/base.c.rej` - ИСПРАВЛЕНО в sultan/fix_base.c.patch
- ✓ `kernel/sys.c.rej` - ИСПРАВЛЕНО в sultan/sys.c_fix.patch

## Использование

### GitHub Actions (автоматически)

Патчи применяются автоматически при запуске workflow:

1. Перейди в Actions → Build and Release Sultan Kernels
2. Выбери "Run workflow"
3. Установи "Include SUSFS?" в `true`
4. Запусти сборку

Патчи будут применены автоматически во время сборки.

### Локальная сборка

Если собираешь локально:

```bash
cd local/android_kernel_google_tensynos

# После интеграции KernelSU-Next и SUSFS
# Применить Sultan патчи
patch -p1 < ../kernel_patches/sultan/fix_base.c.patch
patch -p1 < ../kernel_patches/sultan/sys.c_fix.patch
```

### Проверка статуса

Используй скрипт для проверки:

```bash
cd local/kernel_patches/sultan
./check_patches.sh ../../android_kernel_google_tensynos
```

## Конфигурация

После применения всех патчей, включи в конфигурации ядра:

```
CONFIG_KSU=y
CONFIG_KSU_SUSFS=y
CONFIG_KSU_SUSFS_SUS_MAP=y
CONFIG_KSU_SUSFS_SPOOF_UNAME=y
CONFIG_KSU_SUSFS_TRY_UMOUNT=y
CONFIG_KSU_SUSFS_AUTO_ADD_SUS_BIND_MOUNT=y
CONFIG_KSU_SUSFS_AUTO_ADD_SUS_KSU_DEFAULT_MOUNT=y
CONFIG_KSU_SUSFS_AUTO_ADD_TRY_UMOUNT_FOR_BIND_MOUNT=y
CONFIG_KSU_SUSFS_ENABLE_LOG=y
```

## Отладка

Если в CI появляются .rej файлы, они автоматически собираются и загружаются как артефакты:
- `rej-{codename}-Next-SUSFS.zip` - содержит все .rej файлы и оригинальные файлы

Скачай артефакт из Actions для анализа проблем.

## Дополнительная информация

- KernelSU-Next: https://github.com/tiann/KernelSU
- SUSFS: https://gitlab.com/simonpunk/susfs4ksu
- Патчи SUSFS: `kernel_patches/next/susfs_fix_patches/v2.0.0/`
- Sultan Kernel: https://github.com/kerneltoast/android_kernel_google_tensynos
