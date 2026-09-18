# NYKLIMA OS

Многослойная хобби-ОС для x86_64 на freestanding C. Загружается с собственного
UEFI-загрузчика, без внешних bootloader-зависимостей.

## Состав системы

- **Собственный UEFI-загрузчик** (`uefi/`) — `EFI/BOOT/BOOTX64.EFI`: читает
  `KERNEL.ELF` из FAT16, строит таблицы страниц (identity 4 ГБ + HHDM +
  high-half образа ядра), передаёт ядру framebuffer, карту памяти, RSDP и
  адрес загрузки, после чего передаёт управление `kmain`.
- **Ядро** (`kernel/`):
  - paging: identity 4 ГБ, HHDM, образ ядра 4КБ-страницами;
  - физическая память (bitmap-PMM) + куча free-list;
  - IDT, PIC/IOAPIC, PIT 100 Гц, PS/2 клавиатура и мышь;
  - SMP: ACPI MADT, INIT/SIPI, трамплин real→long mode;
  - кооперативные потоки с sleep/wake;
  - FAT16 (LFN, чтение/запись) поверх ATA PIO;
  - оконный менеджер: drag мышью, taskbar, z-order, backbuffer;
  - программный 3D-рендерер NGL (Z-буфер, текстуры, туман, blend);
  - загрузчик ELF-PIE программ (`pgm`) с API-таблицей ядра.
- **Программы** — SNAKE (`kernel/src/game_snake.c`), спрайты рендерятся
  CUDA-скриптом `tools/render_sprites.py` на хост-GPU и попадают в ОС
  через FAT16 (мост «3050 → FAT16 → NGL»).
- `boot/` — README, попадающий в образ диска.
- `scripts/` — генерация FAT16-образа с MBR и запуск VirtualBox.
- `build.ps1` — сборка ядра, загрузчика, программ и диска.

## Протокол загрузчика (временно Limine-ABI)

Ядро и загрузчик пока общаются структурами протокола Limine
(`kernel/include/limine.h`): загрузчик находит в образе ядра секцию
`.limine_requests` и заполняет ответы (HHDM, framebuffer, memmap, RSDP,
executable address). Это осознанный временный ABI — после стабилизации
загрузки он будет заменён собственной структурой `nyklima_boot_info`.

## Управление

- `F` — файловый менеджер, `I` — система, `G` — 3D-куб, `S` — настройки;
- `Esc` — закрыть активное окно, `Enter` — цикл фокуса;
- `1` — запустить SNAKE (WASD, Esc — выход, Enter — заново);
- мышь: клик по окну/заголовку (drag), двойной клик по файлу — просмотр,
  клик по taskbar — поднять окно.

## Сборка и запуск

```powershell
Set-Location -LiteralPath "$HOME\Documents\операционка"
& .\build.ps1        # сборка ядра, BOOTX64.EFI, образа build\nyklima-uefi.img
& .\build.ps1 -Run   # то же + конвертация в VMDK и запуск VirtualBox
```

Serial-лог ВМ пишется в `build\nyklima-serial.log`. Файловый менеджер
показывает файлы FAT16-диска; записи через `fs_write` сохраняются между
перезагрузками. Пересборка регенерирует образ — пользовательские файлы
на FAT16 стираются.
