# Лабораторная работа: Разработка системы инвентаризации техники «TechInventory»

## 1. Цель работы
Закрепить навыки разработки десктопных приложений на **.NET** с использованием кроссплатформенного UI-фреймворка **Avalonia UI**. Научиться проектировать реляционную базу данных (**MySQL**), выполнять CRUD-операции через **MySqlConnector** и генерировать отчеты в формате Excel с помощью библиотеки **FreeSpire.XLS**.

## 2. Технический стек
*   **Язык:** C# (.NET 6 или .NET 8)
*   **UI Фреймворк:** Avalonia UI (версия 11+)
*   **База данных:** MySQL (локальный сервер или Docker-контейнер)
*   **Драйвер БД:** MySqlConnector (NuGet пакет)
*   **Генерация Excel:** FreeSpire.XLS (NuGet пакет `FreeSpire.XLS`)
*   **Архитектура:** MVVM (рекомендуется) или Code-Behind (для упрощенной версии)

## 3. Структура базы данных
Студент должен создать базу данных `TechInventory` и следующие таблицы. Скрипт создания предоставляется ниже, но студент должен понимать связи.

```sql
-- Справочник должностей
CREATE TABLE Positions (
    Id INT AUTO_INCREMENT PRIMARY KEY,
    Name VARCHAR(100) NOT NULL
);

-- Сотрудники
CREATE TABLE Employees (
    Id INT AUTO_INCREMENT PRIMARY KEY,
    FullName VARCHAR(150) NOT NULL,
    PositionId INT,
    FOREIGN KEY (PositionId) REFERENCES Positions(Id) ON DELETE SET NULL
);

-- Техника (Основные средства)
CREATE TABLE Equipment (
    Id INT AUTO_INCREMENT PRIMARY KEY,
    InvNumber VARCHAR(50) UNIQUE NOT NULL, -- Инвентарный номер
    Name VARCHAR(150) NOT NULL,            -- Наименование (напр. Ноутбук Dell XPS)
    PurchaseDate DATE NOT NULL,
    Cost DECIMAL(10, 2) NOT NULL,
    IsWrittenOff BOOLEAN DEFAULT FALSE,    -- FALSE: на балансе, TRUE: списана
    CurrentEmployeeId INT NULL,            -- Текущий ответственный
    FOREIGN KEY (CurrentEmployeeId) REFERENCES Employees(Id) ON DELETE SET NULL
);

-- История передачи техники
CREATE TABLE Transfers (
    Id INT AUTO_INCREMENT PRIMARY KEY,
    EquipmentId INT NOT NULL,
    FromEmployeeId INT NULL,               -- NULL если первая выдача со склада
    ToEmployeeId INT NOT NULL,
    TransferDate DATETIME NOT NULL,
    FOREIGN KEY (EquipmentId) REFERENCES Equipment(Id),
    FOREIGN KEY (FromEmployeeId) REFERENCES Employees(Id),
    FOREIGN KEY (ToEmployeeId) REFERENCES Employees(Id)
);

-- История списания
CREATE TABLE WriteOffs (
    Id INT AUTO_INCREMENT PRIMARY KEY,
    EquipmentId INT NOT NULL,
    WriteOffDate DATETIME NOT NULL,
    Reason TEXT NOT NULL,                  -- Причина списания
    NeedsDisposal BOOLEAN DEFAULT FALSE,   -- Требуется ли утилизация
    FOREIGN KEY (EquipmentId) REFERENCES Equipment(Id)
);
```

## 4. Функциональные требования

### 4.1. Главное окно и навигация
Приложение должно иметь главное окно с меню или вкладками для переключения между разделами:
1.  **Справочники** (Техника, Сотрудники, Должности).
2.  **Операции** (Передача, Списание).
3.  **Отчеты** (Техника по сотруднику).

### 4.2. Управление справочниками (CRUD)
Для каждой сущности (Техника, Сотрудники, Должности) реализовать:
*   Отображение списка в `DataGrid`.
*   Добавление новой записи (отдельное окно или модальное).
*   Редактирование существующей записи.
*   Удаление записи (с проверкой внешних ключей: нельзя удалить сотрудника, за которым числится техника).
*   **Валидация:** Инвентарный номер должен быть уникальным, стоимость > 0.

### 4.3. Передача техники
*   Отдельное окно «Акт передачи».
*   **Входные данные:** Выбор техники (только та, что на балансе), Выбор сотрудника (кому), Дата.
*   **Логика:**
    1.  При выборе техники автоматически подтягивается текущий владелец (поле «От кого»).
    2.  При сохранении:
        *   Создается запись в таблице `Transfers`.
        *   Обновляется `CurrentEmployeeId` в таблице `Equipment`.
        *   Генерируется Excel-файл «Акт приема-передачи» (ФИО от кого, ФИО кому, Наименование техники, Инв. номер, Дата).
*   Путь сохранения файла выбирает пользователь (`SaveFileDialog`).

### 4.4. Списание техники
*   Отдельное окно «Акт списания».
*   **Входные данные:** Выбор техники, Причина (TextBox), Чекбокс «Требуется утилизация», Дата.
*   **Логика:**
    1.  При сохранении:
        *   Создается запись в `WriteOffs`.
        *   Поле `IsWrittenOff` в `Equipment` устанавливается в `TRUE`.
        *   `CurrentEmployeeId` в `Equipment` обнуляется.
        *   Генерируется Excel-файл «Акт списания» (Техника, Причина, Дата, Статус утилизации).

### 4.5. Отчет по сотруднику
*   Окно с выбором сотрудника из выпадающего списка (`ComboBox`).
*   `DataGrid`, отображающий всю технику, числящуюся за выбранным сотрудником.
*   Кнопка «Экспорт в Excel».
*   Генерируется файл со списком техники (Инв. номер, Наименование, Дата закупки, Стоимость).

## 5. Пример работы с FreeSpire.XLS
Библиотека требует подключения пространства имен `Spire.Xls`. Ниже приведен пример метода для генерации простого документа.

*Установка:* `Install-Package FreeSpire.XLS`

```csharp
using Spire.Xls;

public void GenerateTransferDocument(string filePath, string fromWho, string toWho, string equipmentName, string invNumber, DateTime date)
{
    // 1. Создаем книгу и лист
    Workbook workbook = new Workbook();
    Worksheet sheet = workbook.Worksheets[0];
    sheet.Name = "Акт передачи";

    // 2. Заголовок
    sheet.Range["A1"].Text = "АКТ ПРИЕМА-ПЕРЕДАЧИ ТЕХНИКИ";
    sheet.Range["A1"].Style.Font.IsBold = true;
    sheet.Range["A1"].Style.HorizontalAlignment = HorizontalAlignType.Center;
    
    // 3. Данные (пример заполнения)
    sheet.Range["A3"].Text = $"Дата: {date.ToShortDateString()}";
    sheet.Range["A4"].Text = $"От кого: {fromWho ?? "Склад"}";
    sheet.Range["A5"].Text = $"Кому: {toWho}";
    
    sheet.Range["A7"].Text = "Наименование";
    sheet.Range["B7"].Text = "Инвентарный номер";
    sheet.Range["A7"].Style.Font.IsBold = true;
    sheet.Range["B7"].Style.Font.IsBold = true;

    sheet.Range["A8"].Text = equipmentName;
    sheet.Range["B8"].Text = invNumber;

    // 4. Границы
    sheet.Range["A7:B8"].Style.Borders[BordersLineType.EdgeTop].LineStyle = LineStyleType.Thin;
    sheet.Range["A7:B8"].Style.Borders[BordersLineType.EdgeBottom].LineStyle = LineStyleType.Thin;
    sheet.Range["A7:B8"].Style.Borders[BordersLineType.EdgeLeft].LineStyle = LineStyleType.Thin;
    sheet.Range["A7:B8"].Style.Borders[BordersLineType.EdgeRight].LineStyle = LineStyleType.Thin;

    // 5. Автоподбор ширины
    sheet.AllocatedRange.AutoFitColumns();

    // 6. Сохранение
    workbook.SaveToFile(filePath, ExcelVersion.Version2016);
}
```

## 6. Требования к интерфейсу (Avalonia UI)
1.  **Стили:** Использовать встроенные темы Avalonia (например, `Fluent` или `Simple`) или настроить базовые цвета в `App.xaml`.
2.  **Адаптивность:** Окна не должны «ломаться» при изменении размера (использовать `Grid`, `StackPanel`, `DockPanel`).
3.  **Конфигурация:** Строка подключения к БД не должна быть захардкожена в коде. Использовать файл `appsettings.json` или `config.json`.
