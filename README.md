# supabase-keepalive

对应的supabase仓库是：yuyue92'org - go-project

---

Supabase项目自动保活配置

这是个常见需求，思路很简单：用 GitHub Actions 定时（cron）对 Supabase 数据库做一次真实的读写请求，防止项目因 7 天无活动被 Supabase 自动 pause。

## 1. 建表 SQL（在 Supabase SQL Editor 里执行一次）

```sql
create table if not exists public.keep_alive (
  id serial primary key,
  pinged_at timestamptz not null default now()
);

-- 可选：只保留最近记录，避免表无限增长
create or replace function public.trim_keep_alive()
returns trigger as $$
begin
  delete from public.keep_alive
  where id not in (
    select id from public.keep_alive
    order by pinged_at desc
    limit 20
  );
  return new;
end;
$$ language plpgsql;

drop trigger if exists trim_keep_alive_trigger on public.keep_alive;
create trigger trim_keep_alive_trigger
after insert on public.keep_alive
for each statement
execute function public.trim_keep_alive();

-- 允许通过 anon/service key 写入（如果开了 RLS，需要放开策略）
alter table public.keep_alive enable row level security;

create policy "allow insert for keep alive"
on public.keep_alive
for insert
to anon
with check (true);

create policy "allow select for keep alive"
on public.keep_alive
for select
to anon
using (true);
```

> 如果你用 `service_role` key 来调用，可以不开 RLS 策略，因为 service_role 会绕过 RLS。用 anon key 的话就需要上面这两条策略。

## 2. GitHub Actions workflow：`.github/workflows/supabase-keepalive.yml`

```yaml
name: Supabase Keep Alive

on:
  schedule:
    # 每 3 天执行一次，UTC 时间 03:00（避免刚好卡在 7 天边界上出问题）
    - cron: '0 3 */3 * *'
  workflow_dispatch: {}  # 允许手动触发测试

jobs:
  keep-alive:
    runs-on: ubuntu-latest
    steps:
      - name: Ping Supabase to prevent auto-pause
        run: |
          response=$(curl -s -w "\n%{http_code}" -X POST \
            "${SUPABASE_URL}/rest/v1/keep_alive" \
            -H "apikey: ${SUPABASE_KEY}" \
            -H "Authorization: Bearer ${SUPABASE_KEY}" \
            -H "Content-Type: application/json" \
            -d '{"pinged_at": "now()"}')

          http_code=$(echo "$response" | tail -n1)
          body=$(echo "$response" | sed '$d')

          echo "HTTP status: $http_code"
          echo "Response: $body"

          if [ "$http_code" -ge 400 ]; then
            echo "❌ Keep-alive request failed"
            exit 1
          else
            echo "✅ Keep-alive request succeeded"
          fi
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_KEY: ${{ secrets.SUPABASE_KEY }}
```

## 3. GitHub Secrets 配置

去仓库 `Settings → Secrets and variables → Actions → New repository secret`，添加：

| Secret 名称 | 值 |
|---|---|
| `SUPABASE_URL` | 你的项目 URL，如 `https://xxxxxxxx.supabase.co` |
| `SUPABASE_KEY` | 建议用 `service_role` key（在 Supabase 项目 Settings → API 里找），这样不用配置 RLS 策略；如果安全上不想用 service_role，可以用 `anon` key 但要配上面的 RLS policy |

## 补充说明

- **cron 频率**：Supabase 免费版是"7 天无活动"就 pause，所以 `*/3` 天（每 3 天一次）留了足够余量。你也可以改成 `0 3 * * 0`（每周日一次），但风险稍高（GitHub Actions 的 cron 有时会延迟几分钟到几小时）。
- **用 service_role key 更简单**：不用折腾 RLS，但要注意这个 key 权限很高，只放在 GitHub Secrets 里，不要写死在代码或日志里。
- **workflow_dispatch**：加了手动触发按钮，配置完可以先在 Actions 页面手动跑一次，确认能成功再放心让它定时跑。
- **免费版本身限制**：GitHub Actions 的 scheduled workflow 在仓库 60 天无 push 活动的情况下会被 GitHub 自动禁用，如果你的仓库长期不更新代码，记得偶尔登录看一下 Actions 有没有被暂停。
