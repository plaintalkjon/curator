# Plan Review: Curator_Van_Gogh Bot

## Overall Assessment
The plan is well-structured and comprehensive. The architecture using Supabase Edge Functions is appropriate for this use case. However, there are several technical details that need adjustment before implementation.

## Critical Issues to Address

### 1. Mastodon Client Library Compatibility
**Issue**: The plan uses `mastodon-api` npm package, but Supabase Edge Functions run on Deno, not Node.js.

**Solution Options**:
- **Option A (Recommended)**: Use direct HTTP fetch calls to Mastodon API
- **Option B**: Use a Deno-compatible Mastodon client if available
- **Option C**: Use `npm:` specifier in Deno (e.g., `import Mastodon from 'npm:mastodon-api'`)

**Recommendation**: Use direct fetch calls for simplicity and better Deno compatibility.

### 2. Duplicate Prevention Not Implemented
**Issue**: The plan mentions checking the last 30 entries to avoid duplicates, but the code doesn't implement this.

**Solution**: Add a check before posting:
```typescript
// Check for recent duplicates
const { data: recentPosts } = await supabase
  .from('posts')
  .select('met_object_id')
  .eq('bot_id', bot.id)
  .order('posted_at', { ascending: false })
  .limit(30);
  
const recentIds = new Set(recentPosts?.map(p => p.met_object_id) || []);
// Retry if duplicate found
```

### 3. Row-Level Security (RLS) Setup Missing
**Issue**: The plan mentions RLS but doesn't provide the SQL.

**Solution**: Add this SQL after creating the tables:
```sql
-- Enable RLS on bots table
ALTER TABLE bots ENABLE ROW LEVEL SECURITY;

-- Policy: Only service role can read access_token
CREATE POLICY "Service role can read bots"
  ON bots FOR SELECT
  USING (auth.role() = 'service_role');
```

### 4. Cron Scheduling Setup Missing
**Issue**: The plan mentions Supabase Scheduled Functions but doesn't explain how to set it up.

**Solution**: Supabase cron is configured via `supabase/config.toml` or the dashboard. The function needs to be registered as a cron job.

### 5. Service Role Key Usage
**Issue**: The code uses `SUPABASE_ANON_KEY`, but to read the `access_token` with RLS, you need `SUPABASE_SERVICE_ROLE_KEY`.

**Solution**: Update the Supabase client initialization:
```typescript
const supabase = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')! // Use service role key
);
```

## Implementation Checklist

### Phase 1: Supabase Setup
- [ ] Create Supabase project
- [ ] Enable Edge Functions in project settings
- [ ] Run the SQL schema (tables: bots, schedule, posts)
- [ ] Set up RLS policies
- [ ] Create a test bot row (without real token initially)

### Phase 2: Mastodon Setup
- [ ] Create Mastodon account on mastodon.social
- [ ] Create a new application in Mastodon settings
- [ ] Generate access token with `read` and `write` scopes
- [ ] Test token manually (post a test status)
- [ ] Store token securely in Supabase bots table

### Phase 3: Code Implementation
- [ ] Set up local Supabase development environment
- [ ] Create Edge Function structure
- [ ] Implement MET API fetching logic
- [ ] Implement duplicate prevention
- [ ] Implement Mastodon posting (using fetch or Deno-compatible client)
- [ ] Add error handling and logging
- [ ] Test locally with `supabase functions serve`

### Phase 4: Deployment
- [ ] Deploy Edge Function to Supabase
- [ ] Set environment variables (SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)
- [ ] Configure cron schedule in Supabase
- [ ] Test first manual execution
- [ ] Monitor logs for scheduled runs

## Code Improvements Needed

### 1. Update Mastodon Posting Function
Replace the `mastodon-api` usage with direct fetch calls:

```typescript
async function postToMastodon(
  bot: any,
  imageBuffer: Uint8Array,
  statusText: string
): Promise<string> {
  const instance = bot.mastodon_instance;
  
  // Step 1: Upload media
  const formData = new FormData();
  const blob = new Blob([imageBuffer]);
  formData.append('file', blob, 'artwork.jpg');
  formData.append('description', statusText);
  
  const mediaRes = await fetch(`${instance}/api/v1/media`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${bot.access_token}`,
    },
    body: formData,
  });
  
  if (!mediaRes.ok) {
    throw new Error(`Media upload failed: ${await mediaRes.text()}`);
  }
  
  const media = await mediaRes.json();
  
  // Step 2: Create status
  const statusRes = await fetch(`${instance}/api/v1/statuses`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${bot.access_token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      status: statusText,
      media_ids: [media.id],
      spoiler_text: bot.content_warning || undefined,
    }),
  });
  
  if (!statusRes.ok) {
    throw new Error(`Status creation failed: ${await statusRes.text()}`);
  }
  
  const status = await statusRes.json();
  return status.id;
}
```

### 2. Add Duplicate Prevention
Add before posting:
```typescript
// Check for duplicates in last 30 posts
const { data: recentPosts } = await supabase
  .from('posts')
  .select('met_object_id')
  .eq('bot_id', bot.id)
  .order('posted_at', { ascending: false })
  .limit(30);

const recentIds = new Set(recentPosts?.map(p => p.met_object_id) || []);

// Retry up to 5 times if duplicate
let art;
let attempts = 0;
do {
  art = await getRandomVanGogh();
  attempts++;
  if (attempts > 5) {
    throw new Error('Could not find non-duplicate artwork after 5 attempts');
  }
} while (recentIds.has(art.objectID));
```

### 3. Fix Supabase Client Initialization
```typescript
const supabase = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')! // Changed from ANON_KEY
);
```

## Additional Recommendations

1. **Add a failed_posts table** for better error tracking:
   ```sql
   create table failed_posts (
     id uuid primary key default gen_random_uuid(),
     bot_id uuid references bots(id),
     met_object_id integer,
     error_message text,
     failed_at timestamptz not null default now()
   );
   ```

2. **Add retry logic** for transient failures (MET API, network issues)

3. **Add image validation** - check image size/format before uploading

4. **Consider caching** the MET Van Gogh IDs list in a Supabase table to reduce API calls

5. **Add monitoring/alerting** - set up alerts for consecutive failures

## Next Steps

1. Start with Supabase setup (Phase 1)
2. Get Mastodon credentials (Phase 2)
3. Implement and test locally (Phase 3)
4. Deploy and monitor (Phase 4)

## Questions to Consider

- Do you want to post the same artwork twice per day, or different artworks?
- Should the bot have a profile description/bio?
- Do you want alt text for accessibility (already included in media description)?
- Should there be a way to manually trigger a post for testing?

