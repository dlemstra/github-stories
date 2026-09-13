# The ImageMagick website is now hosted by GitHub

More than two months ago we migrated the ImageMagick website (https://imagemagick.org) from our own servers to GitHub. The migration was a success, and we are now hosting the website on GitHub Pages. This story explains why we made this change and how we did it.

## Why we moved the website to GitHub

The ImageMagick project has been around for over 36 years, and during that time we have had to maintain our own servers to host our website. We have kept the code in a Git repository since 2015, but changes to our website still had to be updated manually on those servers. We could have automated the deployment, but that would require us to maintain an extra script or tool. Because GitHub Pages would do this for us, we decided to move the website there. This way, we can focus on developing ImageMagick and not worry about maintaining our own servers and scripts for our website.

## Difficulties we faced before the migration

Our website was not just a static website. We were using PHP to generate our pages dynamically. It also contained some dynamic content, and our downloads were hosted on our own servers. That made it a bit more complicated to move the website to GitHub Pages. The following paragraphs explain what we had to do for each topic to make the migration successful.

### Preserving our SEO links

We had a lot of links to our website from other websites and search engines. We wanted to make sure that those links would still work after the migration. But our pages were structured like this: https://imagemagick.org/script/[PAGENAME].php. GitHub Pages can serve files with a `.php` extension, but it does not execute PHP. More importantly, a `.php` file is not served as an HTML page that can perform a client-side redirect, so we had to change the structure of our pages. We decided to use the following structure: https://imagemagick.org/[PAGENAME]/. This would make the links a lot cleaner and easier to read. But we also had to make sure that the old links would still work. Because we were using PHP, we could easily move the page to https://imagemagick.org/[PAGENAME]/index.php and create a 301 redirect for the old page. This way, when someone would visit the old page, they would be redirected to the new page. And because we used a 301 redirect, search engines would also update their links to the new page. We started with a single commit where we moved a single page and updated the links in our website to the new URL. We then asked GitHub Copilot to take a look at the commit and create an agent file that we could use to migrate this per page. It created the following agent file: [url-migration.md](url-migration.md). We then used this agent to migrate all our pages to the new structure. This was done in March of 2026 so we could give search engines some time to update their links before we would move our website to GitHub Pages.

### Our forum

Our website also included a forum where people could ask questions and discuss ImageMagick. It was still hosted on our own servers, but we had already locked it down and made it read-only. We had already moved the forum to GitHub Discussions, which is a feature that allows people to ask questions and discuss topics in a more structured way. However, the page was still hosted at https://imagemagick.org/discourse-server/. We had a couple of options. One would be to move the forum to GitHub Pages, but that would require us to create another migration script and fix all the links in it. Another would be to move it entirely to GitHub Discussions, but that would also require quite a lot of work. We decided to move it to a new site on our own server with a different subdomain. We created https://jqmagick.imagemagick.org/discourse-server/ and moved the forum there. This means we will still have this site on our own server, but because it will never be updated again, we will not have to maintain it anymore.

### Our downloads

We also had our downloads hosted on our own servers. These were available at https://imagemagick.org/archive/ and would show a list of all files that were available for download. Because we did not want to host the binaries on GitHub Pages, we decided to move the downloads to a new subdomain on our own server. We created https://download.imagemagick.org/archive/ and moved the downloads there. We also thought about moving the downloads to GitHub Releases and we have now done that but while we were moving the website to GitHub Pages we did not want to include that in the migration. In a future story we will explain how we have done that.

### Anthony Thyssen's usage page

At https://imagemagick.org/Usage/ we had a page that was created by Anthony Thyssen. This page contained a lot of information about how to use ImageMagick. We wanted to make sure that this page would still be available after the migration. We decided to move it to a new subdomain, https://usage.imagemagick.org/, and moved the page there. Because our main website was still using Apache at the time, we could create a 301 redirect for all the pages in the old folder to the new URL:

```
# Redirect /Usage*
RewriteRule ^Usage https://usage.imagemagick.org/ [R=301,L]
```

We initially hosted the https://usage.imagemagick.org/ subdomain on our own server as well. But because the content for this page was already in its own GitHub repository and contained only static HTML, we later realized we could move that subdomain to GitHub Pages instead. This was the first site that we moved to GitHub Pages.

### Contact form

Our website also had a contact form that people could use to contact us. But we did not want to continue that and change the link to GitHub Discussions. The old page is no longer available and we have removed the link from our website. People can now use GitHub Discussions to contact us.

### Dynamic PHP content

Our website had some dynamic content that was generated using PHP. Some of this consisted of small functions that created HTML snippets, while other functions were more complex. We decided to replace the small functions with static HTML snippets and save the more complex functions for later, replacing them with Jekyll layouts during the migration.

## Migrating the website to GitHub Pages

With all the preparations done, we could finally start the migration of our website to GitHub Pages. This started with creating a branch called `pages` and enabling GitHub Pages for that branch. This meant that we could still update our old website while we were migrating the content to GitHub Pages. The migration was done in multiple steps.

### Creating the Jekyll layout

The first step was to create a Jekyll layout that we could use for our pages. Our old pages used a similar template so we could create a single layout that we could use for all our pages. We created a layout called `default.html` and put it in the `_layouts` folder. This layout contained the basic structure of our pages and included the header and footer.

### Using client-side redirects

This was one of the downsides of moving to GitHub Pages. Since we could no longer use server-side redirects like we did with Apache, we had to implement client-side redirects. We did that by creating another layout called `redirect.html` and put it in the `_layouts` folder. It contains the following code:

{% raw %}
```html
{% if page.redirect_to %}
  {% assign dest = page.redirect_to %}
{% else %}
  {% assign dest = page.url | remove: ".php" | replace: "/script/", "/" %}
{% endif %}
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta http-equiv="refresh" content="0; url={{ dest }}">
    <link rel="canonical" href="{{ dest }}">
    <title>Redirecting...</title>
  </head>
  <body>
    <p>Redirecting to <a href="{{ dest }}">{{ dest }}</a>…</p>
    <script>window.location.replace("{{ dest }}");</script>
  </body>
</html>
```
{% endraw %}
We used both the meta refresh and JavaScript methods to make sure that the redirect would work in all browsers. The meta refresh is a fallback for browsers that do not support JavaScript or have it disabled, while the JavaScript method provides a faster redirect for browsers that support it.

### Redirecting old PHP URLs

Even though search engines and users' bookmarks had probably already been updated to point to the new URLs, we still needed to ensure that any old PHP URLs would redirect correctly to the new pages. As mentioned earlier, a `.php` file on GitHub Pages cannot be used as an HTML page for this purpose. However, a small workaround allowed us to create another client-side redirect. Instead of creating a `.php` file for each old URL, we created a directory matching the old PHP filename and placed an `index.html` file inside it that used our redirect layout. GitHub Pages then treats the trailing-slash form of the old URL as a directory and serves that HTML file, allowing the browser to perform the redirect. This way, the old URL would still work and redirect to the new page. We used the same redirect layout for the old forum URL at https://imagemagick.org/discourse-server/. Before we moved imagemagick.org to GitHub Pages, that URL had a 301 redirect to the forum on our own server. Once imagemagick.org itself moved to GitHub Pages, we had to replace that 301 redirect with this client-side redirect as well, even though the forum content itself remains on our own server at https://jqmagick.imagemagick.org/discourse-server/.

### Moving the website

With everything in place, we could finally move the website to GitHub Pages. This meant adding the commits from the `pages` branch to the `main` branch. Once this was done we had to create a `CNAME` file in the root of the repository to specify our custom domain. This ensured that our website would be accessible via our domain rather than the default GitHub Pages URL. We also had to update our DNS settings to point to GitHub Pages. We used an `ALIAS` record for this purpose.

# Conclusion

Migrating the ImageMagick website to GitHub Pages allowed us to take advantage of a modern static site hosting platform while maintaining our custom domain. By using Jekyll layouts and client-side redirects, we were able to replicate much of the functionality of our old PHP-based site. The transition required careful planning and execution, but ultimately resulted in a more maintainable and performant website.
