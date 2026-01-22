function doPost(request) {
  try {
    const { car, day, count, month, year } = request.parameter;

    if (!month || !year) {
      return ContentService.createTextOutput(JSON.stringify({
        success: false,
        error: "Не указан месяц (параметр 'month') или год (параметр 'year')"
      })).setMimeType(ContentService.MimeType.JSON);
    }

    if (!car || !day || day < 1 || day > 31) {
      return ContentService.createTextOutput(JSON.stringify({
        success: false,
        error: "Неверные данные запроса. Требуется 'car', 'day' от 1 до 31, 'month' и 'year'."
      })).setMimeType(ContentService.MimeType.JSON);
    }

    const cellsToColor = count ? parseInt(count, 10) : 1;
    const startDay = parseInt(day, 10);

    if (cellsToColor < 1) {
      return ContentService.createTextOutput(JSON.stringify({
        success: false,
        error: `Количество дней должно быть больше 0.`
      })).setMimeType(ContentService.MimeType.JSON);
    }

    const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
    
    // Получаем информацию о месяцах
    const monthsInfo = getMonthsInfo();
    const currentMonthIndex = monthsInfo.findIndex(m => m.name.toLowerCase() === month.toLowerCase());
    
    if (currentMonthIndex === -1) {
      return ContentService.createTextOutput(JSON.stringify({
        success: false,
        error: `Неизвестный месяц: ${month}`
      })).setMimeType(ContentService.MimeType.JSON);
    }

    const currentMonthInfo = monthsInfo[currentMonthIndex];
    const currentYear = parseInt(year, 10);
    
    // Проверяем, что начальный день не превышает количество дней в месяце
    if (startDay > currentMonthInfo.days) {
      return ContentService.createTextOutput(JSON.stringify({
        success: false,
        error: `День ${startDay} превышает количество дней в месяце ${month} (${currentMonthInfo.days} дней).`
      })).setMimeType(ContentService.MimeType.JSON);
    }

    const sheetName = month + " " + year;
    const sheet = spreadsheet.getSheetByName(sheetName);

    if (!sheet) {
      return ContentService.createTextOutput(JSON.stringify({
        success: false,
        error: `Лист '${sheetName}' не найден`
      })).setMimeType(ContentService.MimeType.JSON);
    }

    const lastRow = sheet.getLastRow();
    const carModels = sheet.getRange(2, 3, lastRow - 1).getValues(); // C2:C
    const backgrounds = sheet.getRange(2, 4, lastRow - 1, currentMonthInfo.days).getBackgrounds(); // D2:AH

    let bookedCar = null;
    let targetCarRowIndex = -1;

    // Находим подходящую машину
    for (let i = 0; i < carModels.length; i++) {
      const fullModel = carModels[i][0];

      if (fullModel.toLowerCase().includes(car.toLowerCase())) {
        let isFree = true;

        // Проверяем доступность на текущем листе
        const daysInCurrentMonth = Math.min(cellsToColor, currentMonthInfo.days - startDay + 1);
        
        for (let j = 0; j < daysInCurrentMonth; j++) {
          const colIndex = startDay - 1 + j;
          const bg = backgrounds[i][colIndex];
          if (bg !== '#ffffff' && bg.toLowerCase() !== 'white') {
            isFree = false;
            break;
          }
        }

        // Если нужно проверить следующий месяц
        if (isFree && cellsToColor > daysInCurrentMonth) {
          const remainingDays = cellsToColor - daysInCurrentMonth;
          const nextMonthInfo = getNextMonthInfo(currentMonthIndex, currentYear, monthsInfo);
          
          if (nextMonthInfo) {
            const nextSheet = spreadsheet.getSheetByName(nextMonthInfo.sheetName);
            if (nextSheet) {
              const nextBackgrounds = nextSheet.getRange(i + 2, 4, 1, remainingDays).getBackgrounds()[0];
              
              for (let j = 0; j < remainingDays; j++) {
                const bg = nextBackgrounds[j];
                if (bg !== '#ffffff' && bg.toLowerCase() !== 'white') {
                  isFree = false;
                  break;
                }
              }
            } else {
              isFree = false; // Следующий лист не найден
            }
          } else {
            isFree = false; // Не можем определить следующий месяц
          }
        }

        if (isFree) {
          bookedCar = fullModel;
          targetCarRowIndex = i;
          break;
        }
      }
    }

    if (!bookedCar) {
      return ContentService.createTextOutput(JSON.stringify({
        success: false,
        error: `Нет доступных машин модели '${car}' с ${startDay} дня ${month} на ${cellsToColor} дней.`
      })).setMimeType(ContentService.MimeType.JSON);
    }

    // Закрашиваем ячейки
    const row = targetCarRowIndex + 2;
    const daysInCurrentMonth = Math.min(cellsToColor, currentMonthInfo.days - startDay + 1);
    
    // Закрашиваем ячейки на текущем листе
    for (let j = 0; j < daysInCurrentMonth; j++) {
      const col = startDay + j + 3; // D=4 → +3
      const cell = sheet.getRange(row, col);
      
      if (j === cellsToColor - 1) {
        cell.setBackground('#00b0f0'); // голубая (последний день)
      } else {
        cell.setBackground('#34a853'); // зелёная
      }
    }

    // Закрашиваем ячейки на следующем листе, если необходимо
    if (cellsToColor > daysInCurrentMonth) {
      const remainingDays = cellsToColor - daysInCurrentMonth;
      const nextMonthInfo = getNextMonthInfo(currentMonthIndex, currentYear, monthsInfo);
      
      if (nextMonthInfo) {
        const nextSheet = spreadsheet.getSheetByName(nextMonthInfo.sheetName);
        if (nextSheet) {
          for (let j = 0; j < remainingDays; j++) {
            const col = j + 4; // Начинаем с колонки D (4)
            const cell = nextSheet.getRange(row, col);
            
            if (daysInCurrentMonth + j === cellsToColor - 1) {
              cell.setBackground('#00b0f0'); // голубая (последний день)
            } else {
              cell.setBackground('#34a853'); // зелёная
            }
          }
        }
      }
    }

    const endDay = startDay + cellsToColor - 1;
    const message = cellsToColor === 1
      ? `Транспорт '${bookedCar}' забронирован на ${startDay} день ${month} ${year}.`
      : `Транспорт '${bookedCar}' забронирован с ${startDay} дня ${month} ${year} на ${cellsToColor} дней.`;

    return ContentService.createTextOutput(JSON.stringify({
      success: true,
      message: message
    })).setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      error: "Ошибка при обработке запроса: " + error.message
    })).setMimeType(ContentService.MimeType.JSON);
  }
}

// Вспомогательная функция для получения информации о месяцах
function getMonthsInfo() {
  return [
    { name: 'January', days: 31 },
    { name: 'February', days: 28 }, // Для простоты не учитываем високосные годы
    { name: 'March', days: 31 },
    { name: 'April', days: 30 },
    { name: 'May', days: 31 },
    { name: 'June', days: 30 },
    { name: 'July', days: 31 },
    { name: 'August', days: 31 },
    { name: 'September', days: 30 },
    { name: 'October', days: 31 },
    { name: 'November', days: 30 },
    { name: 'December', days: 31 }
  ];
}

// Вспомогательная функция для получения информации о следующем месяце
function getNextMonthInfo(currentMonthIndex, currentYear, monthsInfo) {
  let nextMonthIndex = currentMonthIndex + 1;
  let nextYear = currentYear;
  
  if (nextMonthIndex >= monthsInfo.length) {
    nextMonthIndex = 0; // Январь
    nextYear = currentYear + 1;
  }
  
  const nextMonth = monthsInfo[nextMonthIndex];
  return {
    name: nextMonth.name,
    days: nextMonth.days,
    year: nextYear,
    sheetName: nextMonth.name + " " + nextYear
  };
}