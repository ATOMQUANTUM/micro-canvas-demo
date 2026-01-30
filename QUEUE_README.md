Task Queue — Quick Guide

Files
- /home/moltbot/clawd/queue.json — persistent task list (array of tasks)
- /home/moltbot/clawd/queue.log — append-only audit log

Task schema
{
  "id": 1,
  "title": "Short title",
  "description": "Detailed description",
  "created_by": "anton",
  "created_at": "2026-01-30T05:50:00Z",
  "priority": "medium", // low | medium | high
  "status": "open", // open | in-progress | done | blocked
  "owner": null, // "agent:main" or "anton"
  "due": null
}

Telegram commands (use in chat)
- /q add <title> — <short description>  → add a task
- /q list                              → list top open tasks
- /q show <id>                         → show task details
- /q take <id>                         → claim a task (agent will start)
- /q done <id>                         → mark done
- /q help                              → show this help

Behaviour
- When you /q take <id>, the agent marks owner=agent:main and status=in-progress and appends an entry to queue.log with timestamp.
- By default claimed tasks auto-start for short jobs; for credentialed or long jobs agent will ask for confirmation.

Privacy & safety
- No external posting or credential use without explicit approval per task.
- You can request full deletion of the queue files anytime.

Want any extra fields or behaviors? Reply in chat with: "Queue: add <feature>" (e.g., "Queue: add due-date reminders").
