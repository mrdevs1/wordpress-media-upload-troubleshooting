# WordPress Media Upload Troubleshooting Guide

A practical guide to diagnosing and fixing WordPress media upload failures, image-processing errors, PHP upload limits, file permission issues, and missing thumbnails.

## Table of Contents

* [Overview](#overview)
* [Common Errors](#common-errors)
* [1. Check the Upload Process](#1-check-the-upload-process)
* [2. Fix File Size Limit Errors](#2-fix-file-size-limit-errors)
* [3. Fix HTTP Upload Errors](#3-fix-http-upload-errors)
* [4. Fix File Permission Problems](#4-fix-file-permission-problems)
* [5. Troubleshoot Image Processing](#5-troubleshoot-image-processing)
* [6. Fix Missing Thumbnails](#6-fix-missing-thumbnails)
* [7. Check PHP and Server Resources](#7-check-php-and-server-resources)
* [8. Diagnose Plugin and Theme Conflicts](#8-diagnose-plugin-and-theme-conflicts)
* [9. Check WordPress Upload Configuration](#9-check-wordpress-upload-configuration)
* [10. PHP Environment Diagnostic Script](#10-php-environment-diagnostic-script)
* [Troubleshooting Checklist](#troubleshooting-checklist)
* [Security Best Practices](#security-best-practices)
* [Official References](#official-references)

## Overview

WordPress allows administrators and other authorized users to upload images, videos, documents, and other supported files through the Media Library.

Upload failures can originate from several layers:

* WordPress configuration
* PHP upload limits
* File and directory permissions
* PHP memory and execution limits
* GD or Imagick image-processing libraries
* Web server or security rules
* Plugins and themes
* Available disk space and inodes

The correct solution depends on the actual error and server environment. Diagnose the cause before changing production settings.

## Common Errors

| Error or symptom                         | Possible cause                                          |
| ---------------------------------------- | ------------------------------------------------------- |
| The uploaded file could not be moved     | Directory permissions, ownership, or disk issues        |
| File exceeds the maximum upload size     | PHP or hosting upload limits                            |
| HTTP error during upload                 | Server error, timeout, resource limit, or security rule |
| Unable to create image sizes             | GD/Imagick failure, memory exhaustion, or invalid image |
| Thumbnail is missing                     | Image-processing failure or missing generated file      |
| Upload returns 403 Forbidden             | Security rule or access restriction                     |
| Upload returns 500 Internal Server Error | PHP fatal error or server-side failure                  |
| Media Library image returns 404          | Incorrect URL, missing file, or rewrite/CDN issue       |

These are possible causes rather than definitive diagnoses.

## 1. Check the Upload Process

Start with a simple test before making configuration changes.

1. Sign in to WordPress with an account authorized to upload media.
2. Navigate to **Media → Add New Media File**.
3. Upload a small, valid JPG or PNG image.
4. If the upload fails, try the browser uploader if the option is available.
5. Record the complete error message and approximate time of failure.
6. Open the browser's Developer Tools and inspect the failed request in the Network panel.
7. Check the relevant PHP and web server error logs.

The browser uploader can help identify whether the issue is specific to the multi-file uploader, but it will not fix underlying PHP or server configuration problems.

Documentation: https://wordpress.org/documentation/article/media-add-new-screen/

## 2. Fix File Size Limit Errors

WordPress uploads are affected by PHP configuration and any additional limits imposed by the web server or hosting platform.

### Check PHP limits

The following directives commonly affect uploads:

```ini
file_uploads = On
upload_max_filesize = 32M
post_max_size = 40M
memory_limit = 256M
max_execution_time = 120
max_input_time = 120
```

These values are examples, not universal recommendations. Set appropriate values according to your application's needs and hosting plan.

Important considerations:

* `upload_max_filesize` controls the maximum size of an individual uploaded file.
* `post_max_size` limits the overall POST request and should generally be greater than `upload_max_filesize`.
* `memory_limit` affects the memory available to PHP, including image processing.
* Execution and input time limits may affect large or slow requests.

### On cPanel hosting

1. Open **cPanel → MultiPHP INI Editor**, if available.
2. Select the correct domain.
3. Review the PHP directives.
4. Change only the necessary settings within the limits permitted by your hosting provider.
5. Retry the upload.

If you cannot edit the settings, contact your hosting provider.

Do not assume that editing `.htaccess` or `wp-config.php` will override PHP settings on every hosting platform.

## 3. Fix HTTP Upload Errors

A generic HTTP error does not identify a single cause.

### Diagnostic steps

1. Open the browser's Developer Tools.
2. Select the **Network** tab.
3. Retry the upload.
4. Locate the failed request.
5. Record its HTTP status code and response.
6. Check PHP-FPM, Apache, LiteSpeed, or Nginx logs as appropriate.
7. Check firewall or ModSecurity events at the same timestamp.

### Common HTTP statuses

* **400:** The request may be malformed or rejected by application logic.
* **403:** Access may be blocked by permissions or a security rule.
* **404:** The requested endpoint may be unavailable.
* **413:** A request or body-size limit may have been exceeded.
* **500:** A server-side error occurred.
* **502 or 504:** An upstream service may have failed or timed out.

Actual causes vary by server configuration. A proxy may also return a status before the request reaches WordPress.

### LiteSpeed and Cloudflare

For websites using LiteSpeed or Cloudflare:

* Review relevant server and firewall logs.
* Check whether a security rule is blocking the upload request.
* Inspect request-size restrictions and timeout settings.
* Check whether the error is generated by the origin server or proxy.
* Avoid disabling security features globally.

If a rule is confirmed to be responsible, make the narrowest appropriate correction and retest.

## 4. Fix File Permission Problems

WordPress needs appropriate access to its upload directory to save media files and generate image sizes.

The usual upload directory is:

```text
wp-content/uploads/
```

### Check the directory

Using cPanel File Manager or authorized SSH access:

1. Locate the WordPress installation.
2. Open `wp-content/uploads/`.
3. Check whether the expected year/month directory exists.
4. Verify directory ownership and permissions.
5. Check whether the filesystem is read-only or out of space.
6. Review the PHP error log for permission-related messages.

### Typical permission guidance

Common starting points are:

* Directories: `755`
* Files: `644`

These are not universal requirements. The correct settings depend on the hosting environment and PHP execution model.

Do not recursively change ownership or permissions across the entire account without verifying the correct user and group. Avoid `777` permissions as a workaround.

## 5. Troubleshoot Image Processing

WordPress may successfully receive an uploaded image but fail when generating thumbnails or other image sizes.

### Check GD and Imagick

WordPress commonly uses the GD or Imagick image-processing library when available.

Check the PHP extensions:

```bash
php -m | grep -Ei '^(gd|imagick)$'
```

On cPanel or CloudLinux, the command-line PHP version may differ from the PHP version used by the website. Confirm the extensions for the domain's active PHP version.

You can also check the site's **Tools → Site Health → Info** screen for relevant server information.

### Common image-processing problems

* GD or Imagick is not available to the active PHP version.
* PHP runs out of memory while processing a large image.
* The image is corrupt or uses an unsupported encoding.
* The image-processing library encounters a resource limit.
* The uploaded file is not a valid image despite its extension.
* File permissions prevent WordPress from writing generated sizes.

### Recommended checks

1. Test with a small, valid JPEG or PNG image.
2. Review PHP logs at the time of the failed upload.
3. Confirm that GD or Imagick is available.
4. Check PHP memory usage and server resource limits.
5. Test another image to determine whether the problem is file-specific.
6. Update or configure the image-processing library only after confirming the cause.

Avoid installing or removing PHP extensions on a production server without checking compatibility and the impact on other websites.

## 6. Fix Missing Thumbnails

If an image uploads successfully but its thumbnails are missing, determine whether the files were generated and whether WordPress can access them.

### Troubleshooting steps

1. Open the affected media item in WordPress.
2. Check whether the original image loads.
3. Inspect the upload directory for the original file and generated image sizes.
4. Review PHP logs for image-processing errors.
5. Check the configured image sizes under **Settings → Media**.
6. Review whether a theme or plugin registers additional image sizes.
7. Regenerate thumbnails using a trusted, maintained tool if the files are missing.

Before regenerating thumbnails in bulk, back up the site and verify that sufficient disk space is available.

Do not delete original images as part of a thumbnail repair unless you have verified a separate, recoverable backup.

## 7. Check PHP and Server Resources

Large images can require substantially more memory during processing than their compressed file size suggests.

### Check available disk space

On a Linux server with authorized shell access:

```bash
df -h
```

Check inode usage:

```bash
df -i
```

A filesystem can run out of inodes even when it still has free storage capacity.

### Review logs

Depending on the hosting environment, relevant logs may include:

* WordPress `wp-content/debug.log`, if debugging is enabled
* PHP or PHP-FPM error logs
* Apache or LiteSpeed error logs
* Nginx error logs
* Hosting control panel resource usage logs
* ModSecurity or web application firewall audit logs

Use the logs applicable to your environment and correlate entries with the failed upload's timestamp.

### Check resource limits

Review:

* PHP memory limit
* Hosting account memory and process limits
* PHP-FPM worker capacity
* Available disk space and inodes
* Web server request-size limits
* Relevant request timeouts

On CloudLinux or other resource-managed hosting, account-level limits may apply even if the server has free memory or CPU capacity.

Do not increase every limit simultaneously. Change only the setting supported by the diagnostic evidence.

## 8. Diagnose Plugin and Theme Conflicts

Plugins and themes can modify uploads, image sizes, metadata, and image-processing behavior.

### Safe testing procedure

1. Back up the website and database.
2. Reproduce the problem on a staging copy.
3. Record the current theme, active plugins, PHP version, and error logs.
4. Test with suspected plugins disabled one at a time.
5. Test with a standard theme if appropriate.
6. Repeat the upload after each change.
7. Restore the original configuration.
8. Confirm the suspected conflict before changing the production site.

Pay particular attention to:

* Image optimization plugins
* WebP conversion plugins
* CDN integrations
* Media library management plugins
* Security plugins
* Plugins that register custom image sizes
* Custom code in themes or must-use plugins

Do not deactivate security protections on a live production site without an appropriate test plan.

## 9. Check WordPress Upload Configuration

WordPress normally stores uploads under:

```text
wp-content/uploads/
```

The Media settings screen controls options such as organizing uploads into year- and month-based directories.

Review **Settings → Media** and verify that any custom upload configuration matches the actual filesystem.

If the installation uses custom constants, a custom uploads directory, or a multisite configuration, inspect those settings before changing paths.

Do not manually modify database options or define custom upload constants just to suppress an error. First identify why WordPress is using the wrong location or cannot write to the configured one.

## 10. PHP Environment Diagnostic Script

The following standalone script reports selected PHP settings and the availability of common image-processing extensions.

It does not upload files, modify WordPress, or change server settings.

Save it locally as `media-environment-check.php`:

```php
<?php
/**
 * Basic PHP media environment diagnostic.
 *
 * Run from the command line:
 * php media-environment-check.php
 */

$settings = array(
    'PHP version'          => PHP_VERSION,
    'file_uploads'         => ini_get( 'file_uploads' ),
    'upload_max_filesize'  => ini_get( 'upload_max_filesize' ),
    'post_max_size'        => ini_get( 'post_max_size' ),
    'memory_limit'         => ini_get( 'memory_limit' ),
    'max_execution_time'   => ini_get( 'max_execution_time' ),
    'max_input_time'       => ini_get( 'max_input_time' ),
    'GD available'         => extension_loaded( 'gd' ) ? 'Yes' : 'No',
    'Imagick available'    => extension_loaded( 'imagick' ) ? 'Yes' : 'No',
);

foreach ( $settings as $name => $value ) {
    printf( "%-24s %s\n", $name . ':', (string) $value );
}
```

Run it from the command line using the PHP binary relevant to the environment:

```bash
php media-environment-check.php
```

**Important:** Command-line PHP settings may differ from the settings used by PHP-FPM or the web server. Treat the output as a preliminary diagnostic, not proof of the website's active configuration.

Run the script locally or through an authorized terminal. If you temporarily place diagnostic scripts on a web server, restrict access and remove them immediately after use.

## Troubleshooting Checklist

* [ ] Record the exact upload error.
* [ ] Test with a small, valid image.
* [ ] Inspect the failed browser request.
* [ ] Check PHP upload and POST size limits.
* [ ] Verify upload directory ownership and permissions.
* [ ] Check disk space and inode availability.
* [ ] Confirm GD or Imagick availability.
* [ ] Review PHP and web server error logs.
* [ ] Check resource limits and security events.
* [ ] Test plugin and theme conflicts on staging.
* [ ] Verify original images and generated thumbnails.
* [ ] Retest after the fix.
* [ ] Remove temporary diagnostic files and debug logs when no longer needed.

## Security Best Practices

* Keep WordPress, plugins, themes, and PHP updated.
* Upload only permitted file types from trusted sources.
* Do not disable upload validation to bypass errors.
* Avoid world-writable permissions such as `777`.
* Never expose PHP errors or diagnostic output publicly.
* Do not publish database credentials, API tokens, or customer files.
* Back up original media before bulk repair operations.
* Test potentially disruptive changes on staging first.
* Remove temporary diagnostic scripts after use.

## Official References

* [WordPress Media Add New Screen](https://wordpress.org/documentation/article/media-add-new-screen/)
* [WordPress Media Library Screen](https://wordpress.org/documentation/article/media-library-screen/)
* [WordPress Settings Media Screen](https://wordpress.org/documentation/article/settings-media-screen/)
* [WordPress Use Images](https://wordpress.org/documentation/article/use-images/)
* [WordPress Debugging](https://developer.wordpress.org/advanced-administration/debug/debug-wordpress/)
* [PHP File Uploads](https://www.php.net/manual/en/features.file-upload.php)
* [PHP Configuration Directives](https://www.php.net/manual/en/ini.core.php)

## Contributing

Contributions are welcome.

When submitting a troubleshooting improvement:

1. Describe the symptom and affected environment.
2. Include sanitized error messages and relevant version information.
3. Explain how the root cause was identified.
4. Document the verified fix and any limitations.
5. Avoid including credentials, private customer information, or production secrets.

## License

This repository can be distributed under the MIT License. Add an MIT `LICENSE` file to the repository if you choose this license.

---

**Disclaimer:** This is an independent troubleshooting resource and is not an official WordPress project. Always verify the root cause and back up production systems before applying potentially disruptive changes.
