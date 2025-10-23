{
"max_tokens": 16384,
"temperature": 0.0,
"model": "gpt-4.1",
  "messages": [
    {
      "role": "system",
      "content": "{{bpmn.proc(Prompt):sanitizeForJson}}"
    },
    {
      "role": "user",
      "content": "Вот последние сообщения клиента\n\"{{lead.dialogMessages:sanitizeForJson}}\"\n"
    }
  ]
}