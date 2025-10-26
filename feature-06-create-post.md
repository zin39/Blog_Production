# Feature #6: Create Blog Post - Design Document

## 1. Overview

### What are we building ?

[We are building a blog post editor which allows writting markdown content, uploading images , selecting categorieg
with the ability to save drafts or publish immediately.]

### Why are we building it ?

[ To enable creating and publishing technical content, allowing me to share knowledge and document projects on my own
platform.]

## 2. Requirements Summary

### Functional Requirements

**Form Fields:**

- Title input (max 100 characters, required)
- Description textarea (max 300 characters, required)
- Category dropdown (required, populated from categories table)
- Content markdown editor (max 50,000 characters, required)
- Image upload (max 5MB per file, multiple files allowed, formats: jpeg, png, gif, webp)

**Save Behaviour:**

- Auto Save every 20 seconds (silent, no user action required)
- Manual "Save draft" button (shows toast: "Draft saved")
- Manual "Publish " button ( shows confirmation modal, then publishes)

**Slug Generation:**

- Auto generated from title on first save
- Format: lowercase, hypens for spaces, remove special characters
- If duplicate exists, append '-{unique_id}'
- **Locked after publish** (never changes, even if title is edited)

**Draft vs Publshed:**

- New posts start as drafts (status = 'draft')
- Drafts can be edited freely, slug can change
- Published posts: slug locked, content can still be edited
- Draft -> Publish: sets status = 'published', published_at timestamp

**Image Handling:**

- Uploaded to filesystem: '/uploads/{year}/{month}/{uuid}--{filename}'
- URL stored in database
- User copies URL to insert in markdown

### Non-Functional Requirements

**Security:**

- Aunthentication required( session-based, validated on every request)
- Server-side markdwon -> HTML conversion using 'marked'
- HTML sanitization using 'sanitize-html' (remove script tags, dangerous attributes)
- Client-side sanitization using 'DOMPurify' before rendering
- Image uplaod validation (filetype, file size)
- File rename with UUID (never trust client)

**Performance:**

- Auto-save must not block UI (async operation)
- Image upload progress indicator
- Form validation on client-side (immediate feedback)
- Server-side validation (never trust client)

**User Experience:**

- Auto-save preserves work (max 20 seconds of data loss)
- Clear validation errors ( inline on form fields)
- Success/error feedback (toasts, modals)
- After publish: redirect to admin page

### Edge Cases Handled

| Scenario                          | Behaviour                                      |
| --------------------------------- | ---------------------------------------------- |
| Title empty on publish            | Show error: "Title Required"                   |
| Context exceeds 50,000 chars      | Show error at character limit                  |
| User navigates away while editing | Auto-save preserves work                       |
| Browser crashes                   | Last auto-save preserved (max 20s loss)        |
| Duplicate slug                    | Append '-{id}' (e.g, '/learn-react-2')         |
| Edit published post title         | Title updates, **slug stays locked**           |
| Upload files >5MB                 | Show error: "File too large (max 5MB)"         |
| Invalid file type                 | Show error: "Only jpg, png, gif, webp allowed" |
| Unauthorized user                 | 401 Unauthorized (auth middleware blocks)      |
| Network failure during save       | Show retry option, preserve local draft        |

## 3. Database Schema

### Posts Table

| Column Name  | Data Type                  | Constraints                        | Reasoning                                      |
| ------------ | -------------------------- | ---------------------------------- | ---------------------------------------------- |
| id           | uuid                       | primary key                        | Unique identifier for each post                |
| title        | varchar(100)               | Not Null                           | Post title, max 100 chars per requirements     |
| description  | varchar(300)               | Not null                           | Short preview shown on homepage, max 300 chars |
| content      | TEXT                       | Not Null                           | Post Conent is markdown, no size limit         |
| slug         | varchar(255)               | Not null Unique                    | URL-safe identifier, must be unique            |
| category_id  | uuid                       | Not null references categories(id) | Foreign key to categories table                |
| status       | enum ("draft","published") | Not null default 'draft'           | post publication state                         |
| images       | json                       | Null                               | Array of image URL's stored as JSON            |
| created_at   | Timestamp                  | Not null default now()             | When post was first created                    |
| updated_at   | Timestamp                  | Not null default now()             | Last modification time                         |
| published_at | Timestamp                  | Null                               | When post was published(null for drafts)       |

### Indexes

- `CREATE INDEX idx_posts_slug ON posts(slulg);` - Fast lookup by URL
- `CREATE INDEX idx_posts_category_id ON posts(category_id);` - Filter by category
- `CREATE INDEX idx_posts_status ON posts(status);` - Filter draft vs published
- `CREATE INDEX idx_posts_published_at ON posts(published_at);` - Sort by publish date
