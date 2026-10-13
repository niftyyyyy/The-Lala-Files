# Sword of Power — CC300 capstone

- `build-plan.md` — Build Plan v3 (simplified), the source of truth.
- `chat.template.html` — the prop chat page; `__PLAN__` is replaced with the plan.
- `chat.html` — built page (published as a claude.ai artifact).

Rebuild after editing the plan:

    python3 -c "p=open('build-plan.md').read(); s=open('chat.template.html').read().replace('__PLAN__',p); open('chat.html','w').write(s)"
