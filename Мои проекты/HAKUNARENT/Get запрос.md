function doGet(request) {
  const { car, month, day, count, year } = request.parameter;

  if (!car) {
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      error: "Не указана модель (параметр 'car')"
    })).setMimeType(ContentService.MimeType.JSON);
  }

  if (!month) {
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      error: "Не указан месяц (параметр 'month')"
    })).setMimeType(ContentService.MimeType.JSON);
  }

  if (!year) {
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      error: "Не указан год (параметр 'year')"
    })).setMimeType(ContentService.MimeType.JSON);
  }

  if (!day || !count) {
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      error: "Не заданы 'day' (день начала) и 'count' (количество дней)"
    })).setMimeType(ContentService.MimeType.JSON);
  }

  // 🔹 Проверка на корректность даты
  const inputMonthIndex = getMonthIndex(month);
  const inputYear = parseInt(year, 10);
  const now = new Date();
  const currentMonthIndex = now.getMonth(); // 0-11
  const currentYear = now.getFullYear();

  if (
    inputYear < currentYear ||
    (inputYear === currentYear && inputMonthIndex < currentMonthIndex)
  ) {
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      message: "Некорректная дата"
    })).setMimeType(ContentService.MimeType.JSON);
  }
  // 🔹 Конец добавленной проверки

  const startDay = parseInt(day, 10);
  const daysCount = parseInt(count, 10);
  const startYear = parseInt(year, 10);

  if (startDay < 1 || startDay > 31 || daysCount < 1) {
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      error: `Некорректный диапазон дней: с ${startDay} на ${daysCount} дней`
    })).setMimeType(ContentService.MimeType.JSON);
  }

  const results = getCarAvailabilityMultiMonth(car, month, startDay, daysCount, startYear);

  // если нет свободных машин
  if (Object.keys(results).length === 0) {
    return ContentService.createTextOutput(JSON.stringify({
      message: "Нет свободных"
    })).setMimeType(ContentService.MimeType.JSON);
  }

  return ContentService.createTextOutput(JSON.stringify(results))
    .setMimeType(ContentService.MimeType.JSON);
}
