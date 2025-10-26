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

### Categories Table

| Column Name   | Data Type    | Constraints            | Reasoning                                     |
| ------------- | ------------ | ---------------------- | --------------------------------------------- |
| id            | UUID         | Primary Key            | Unique identifier for each category           |
| category_name | varchar(100) | Unique Not null        | Category name, max 100 chars                  |
| category_slug | varchar(255) | Unique Not null        | Autogenerated from category_name              |
| Description   | varchar(255) | null                   | Short preview shown on category               |
| Created_at    | timestamp    | not null default now() | When category was first created               |
| Updated_at    | timestamp    | not null default now() | Last modification time, updates on any change |

### Indexes

- `CREATE INDEX idx_categories_category_slug ON categories(category_slug);` - Fast lookup by slug_name

## 4. API Endpoints

### POST /api/posts

**Purpose:** Create a new blog post (draft or published )

**Auth Required:** Yes (session-based authentication)

**Request Body:**

```json
{
  "title": "How to Learn React",
  "description": "A Comprehensive guide to learning React in 2025",
  "category_id": "550e8400-e29b-41d4-a716-44665540000",
  "content": "# Introduction\n\nReact is ...",
  "status": "draft",
  "images": [
    "/uploads/2025/10/abc123-diagram.png",
    "/uploads/2025/10/xyz789-screenshot.png"
  ]
}
```

**Validation Rules :**

- `title` : Required, max 100 chars,
- `description` : Required, max 300 chars,
- `category_id` : Required, must exist in categories table
- `status` : Required, enum ('draft' or 'published')
- `images ` : Optional, array of strings

**Response (201 Created):**

```json
{
    "success": true,
    "data" : {
        "id" : "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "title" : "How to Learn React",
        "description" : "A Comprehensive guide to learning React in 2025",
        "slug" : " how-to-learn-react",
        "category_id" : "550e8400-e29b-414d4-a716-446655440000",
        "content" : "# Introduction\n\nReact is ...",
        "status" : "draft",
        "images" : [...],
        "created_at": "2025-10-25t10:30:00Z",
        "updated_at": "2025-10-25T10:30:00Z",
        "published_at" : null
    }
}
```

**Error Response:**

- **400 Bad Request:**

```json
{
  "success": false,
  "error": "Validation failed",
  "details": {
    "title": "Title is required",
    "content": "Content exceeds 50,000 characters"
  }
}
```

- **401 Aunauthorized:**

```json
{
  "success": false,
  "error": "Authentication required"
}
```

- **404 Not Found:**

```json
{
  "success": false,
  "error": "Category not found"
}
```

- **500 Internal Server Error:**

```json
{
  "success": false,
  "error ": "Failed to create post"
}
```

**Processing Steps:**

1. Validate session authentication
2. Validate request body fields
3. Verify category_id exists
4. Sanitize content (markdown -> HTML -> Sanitize)
5. Generate slug from title (handle duplicates)
6. Insert into database
7. Return created post
