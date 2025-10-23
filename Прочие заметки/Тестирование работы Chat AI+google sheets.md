Задача на текущий момент: Проверить работу Chatter AI, где данные будут подтягиваться из Google Sheets через механизм развёртывания скрипта как веб приложения.


Предполагаемый код, который обеспечивает такой функционал:
```
function handleRequest(data) {
  const sheetName = data.sheetName;
  const searchColumn = data.searchColumn;
  const searchValue = data.searchValue;

  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  let sheets = [];

  if (sheetName) {
    const sheet = spreadsheet.getSheetByName(sheetName);
    if (!sheet) {
      return ContentService.createTextOutput(JSON.stringify({ error: 'Sheet not found' }))
        .setMimeType(ContentService.MimeType.JSON);
    }
    sheets.push(sheet);
  } else {
    sheets = spreadsheet.getSheets();
  }

  if (searchColumn && !searchValue) {
    let results = [];

    sheets.forEach(sheet => {
      try {
        const range = sheet.getDataRange();
        const values = range.getValues();
        const headers = values[0];

        const idIndex = headers.indexOf(searchColumn);
        if (idIndex === -1) {
          return;
        }

        let columnValues = values.slice(1).map(row => {
          return { [searchColumn]: formatCellValue(row[idIndex]) };
        });

        results = results.concat(columnValues);
      } catch (error) {
        console.error(`Error processing sheet: ${sheet.getName()}, Error: ${error.message}`);
      }
    });

    return ContentService.createTextOutput(JSON.stringify({ results: results }))
      .setMimeType(ContentService.MimeType.JSON);
  }

  if (!searchValue) {
    return ContentService.createTextOutput(JSON.stringify({ error: 'Missing search value' }))
      .setMimeType(ContentService.MimeType.JSON);
  }

  // Задаем в промте разделитель для поиска при нескольких значениях
  const searchPhrases = searchValue.split('%%').map(value => value.trim().toLowerCase().split(' '));

  if (!searchPhrases || searchPhrases.length === 0) {
    return ContentService.createTextOutput(JSON.stringify({ error: 'One or more required parameters are missing' }))
      .setMimeType(ContentService.MimeType.JSON);
  }

  let foundRows = [];

  sheets.forEach(sheet => {
    try {
      const range = sheet.getDataRange();
      const values = range.getValues();
      const headers = values[0];

      let dataArray = values.slice(1).map(row => {
        let rowObject = {};
        headers.forEach((header, index) => {
          rowObject[header] = formatCellValue(row[index]);
        });
        return rowObject;
      });

      let sheetFoundRows;
      if (searchColumn) {
        const idIndex = headers.indexOf(searchColumn);
        if (idIndex === -1) {
          return;
        }
        sheetFoundRows = dataArray.filter(item => {
          let cellValue = item[searchColumn].toString().toLowerCase();
          return searchPhrases.some(phrase => phrase.every(word => cellValue.includes(word)));
        });
      } else {
        sheetFoundRows = dataArray.filter(item => {
          return searchPhrases.some(phrase => {
            return Object.values(item).some(value => {
              return phrase.every(word => value.toString().toLowerCase().includes(word));
            });
          });
        });
      }

      foundRows = foundRows.concat(sheetFoundRows);
    } catch (error) {
      console.error(`Error processing sheet: ${sheet.getName()}, Error: ${error.message}`);
    }
  });

  if (foundRows.length > 0) {
    return ContentService.createTextOutput(JSON.stringify({ results: foundRows }))
      .setMimeType(ContentService.MimeType.JSON);
  } else {
    return ContentService.createTextOutput(JSON.stringify({ error: 'Values not found' }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function formatCellValue(value) {
  if (value instanceof Date) {
    return Utilities.formatDate(value, Session.getScriptTimeZone(), 'dd.MM.yyyy HH:mm:ss');
  }
  return value;
}

function doGet(e) {
  return handleRequest(e.parameter);
}

function doPost(e) {
  if (!e.postData || !e.postData.contents) {
    return ContentService.createTextOutput(JSON.stringify({ error: 'Invalid request' }))
      .setMimeType(ContentService.MimeType.JSON);
  }

  let data;
  try {
    data = JSON.parse(e.postData.contents);
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({ error: 'Invalid JSON' }))
      .setMimeType(ContentService.MimeType.JSON);
  }

  return handleRequest(data);
}
```