---
name: url-migration
tools: ["read", "edit", "search", "execute"]
---
# Agent Instructions: URL Migration for ImageMagick Website

## Purpose
This agent file provides instructions for migrating pages from the `/script/` directory to their own dedicated directories with cleaner URLs.

## Migration Pattern (Based on defines.php Migration)

### Overview
The migration follows these steps:
1. Create a new directory with the desired URL name
2. Move page content to the new directory's `index.php`
3. Convert old file to a 301 redirect
4. Update all references throughout the website

### Step-by-Step Process

#### Step 1: Create New Directory Structure
For a page currently at `script/PAGENAME.php`, create:
```
PAGENAME/
  └── index.php
```

The new `index.php` should contain:
```php
<?php
  $title='Page Title';
  $description='Page description for SEO';
  $folder='PAGENAME';
  include('../script/session.php');
?>
```

#### Step 2: Convert Old File to Redirect
Replace the content of `script/PAGENAME.php` with:
```php
<?php
  header("Location: /PAGENAME/", true, 301);
  exit();
?>
```

#### Step 3: Update All References
Search for and replace all references to the old URL pattern:

**Old patterns to find:**
- `<?php echo $_SESSION['RelativePath']?>/../script/PAGENAME.php`
- Relative paths like `../script/PAGENAME.php`
- `href="<?php echo $_SESSION['RelativePath']?>/../script/PAGENAME.php"`
- Direct hrefs like `href="PAGENAME.php"`
- Direct hrefs like `href="/script/PAGENAME.php"`

**Replace with:**
- `/PAGENAME/`
- `href="/PAGENAME/"`

### Patterns to Search For
Look for these patterns in the codebase:
- `<a href="<?php echo $_SESSION['RelativePath']?>/../script/PAGENAME.php">`
- `<a href="../script/PAGENAME.php">`
- `<a href="PAGENAME.php">`
- `<a href="/script/PAGENAME.php">`
- Any similar relative path constructions

### Example: defines.php Migration

**Files Changed:**
1. Created: `defines/index.php` (new location)
2. Modified: `script/defines.php` (now redirects)
3. Updated references in:
   - `include/command-line-options.php`
   - `include/formats.php`

**Changes Made:**
- From: `<?php echo $_SESSION['RelativePath']?>/../script/defines.php`
- To: `/defines/`

## Agent Execution Instructions

When asked to migrate a URL, follow these steps:

1. **Identify the page**: Ask which script file to migrate (e.g., `script/PAGENAME.php`)

2. **Extract metadata**: Read the current script file to get:
   - `$title` variable
   - `$description` variable
   - Any other specific variables

3. **Search for references**: Use the `search` tool to find all files referencing the old URL pattern `script/PAGENAME.php`

4. **Create new structure**:
   - Create new directory `PAGENAME/`
   - Create `PAGENAME/index.php` with proper metadata and folder variable
   - Ensure `$folder` variable matches directory name

5. **Convert old file**: Replace content with 301 redirect

6. **Update all references**: 
   - Replace all old URL patterns with `/PAGENAME/`
   - Always use `multi_replace_string_in_file` when updating more than one file — batch all replacements into a single call rather than calling `replace_string_in_file` repeatedly

7. **Verify**: Check if there are any remaining references to old URL

8. **Add new file to git**: Run `git add PAGENAME/index.php` to stage the new file

## URL Pattern Standards

### Old Pattern (Deprecated)
```php
<a href="<?php echo $_SESSION['RelativePath']?>/../script/PAGENAME.php">
```

### New Pattern (Preferred)
```php
<a href="/PAGENAME/">
```

## Common Files to Check for References

Based on the defines migration, commonly updated files include:
- `include/*.php` - All include files
- Other script files in `script/`
- Main page files in root
- README and documentation files

## Validation Checklist

After migration, verify:
- [ ] New directory and index.php created
- [ ] Old script file contains 301 redirect
- [ ] All references updated (use grep to verify)
- [ ] No broken links remain
- [ ] Redirect works correctly
- [ ] SEO metadata preserved

## Notes

- Always use 301 (permanent) redirect, not 302 (temporary)
- Preserve all SEO metadata ($title, $description)
- Use absolute URLs (`/PAGENAME/`) not relative paths
- Include trailing slash in new URLs (`/PAGENAME/` not `/PAGENAME`)
- The `$folder` variable should match the directory name


