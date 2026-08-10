# Jeel Pay for WordPress and WooCommerce

When a user runs WooCommerce, prefer the official **JeelPay for WooCommerce** plugin over building a
custom checkout API integration. It is available from the
[WordPress Plugin Directory](https://wordpress.org/plugins/jeelpay-for-woocommerce/).

## Prerequisites

| Component | Requirement |
| --- | --- |
| WordPress | 5.8 or newer |
| WooCommerce | 5.0 or newer and activated |
| PHP | 7.4 or newer |
| Transport | Valid SSL certificate; HTTPS is required |
| Access | WordPress administrator access |

## Recommended Installation

1. Open **Plugins > Add New** in WordPress admin.
2. Search for **JeelPay for WooCommerce**.
3. Select **Install Now**, then **Activate Plugin**.
4. Open **WooCommerce > Settings > Payments** and confirm JeelPay appears.
5. Open the JeelPay payment settings, enter the credentials supplied by Jeel Pay, and enable sandbox
   mode for initial validation.
6. Complete a sandbox transaction before switching to production credentials and disabling sandbox
   mode.

Do not ask WooCommerce users to paste API credentials into theme code, frontend JavaScript, or custom
snippets. Use the plugin settings and keep credentials server-side.

## Manual Installation

If WordPress admin installation is unavailable:

1. Download the current ZIP from the official WordPress Plugin Directory.
2. Extract and upload the `jeelpay-for-woocommerce` directory to `/wp-content/plugins/` using SFTP/FTP,
   or upload and extract the ZIP with the hosting control panel.
3. Activate **JeelPay for WooCommerce** under **Plugins > Installed Plugins**.
4. Configure it under **WooCommerce > Settings > Payments > JeelPay**.

Use manual installation only when the normal WordPress installer cannot be used. Do not source the
plugin from unofficial download sites.

## Verification

- Confirm the plugin is active.
- Confirm JeelPay appears under WooCommerce payment settings.
- Verify WooCommerce itself is active.
- Check the storefront checkout and browser console for errors.
- Test the complete checkout in sandbox mode over HTTPS.
- Switch only credentials and sandbox/production mode when going live; do not customize endpoint URLs
  unless the plugin explicitly exposes a supported setting.

## Troubleshooting

### Upload exceeds `upload_max_filesize`

Use SFTP/FTP manual installation or ask the hosting provider to raise the upload limit.

### Plugin folder already exists

An earlier copy exists under `/wp-content/plugins/jeelpay-for-woocommerce/`. Back up any intentional
local customizations, then use WordPress's normal update/reinstall process. Do not delete a live plugin
directory without confirming how the site is deployed and how it will be restored.

### Missing required plugin: WooCommerce

Install and activate WooCommerce first, then activate JeelPay for WooCommerce.

### HTTP error during upload

Use SFTP/FTP or the hosting file manager. If the host is memory constrained, coordinate an appropriate
PHP memory-limit change with the site administrator instead of blindly editing production config.

### JeelPay does not appear as a payment method

Confirm all prerequisites, verify that both plugins are active, review WooCommerce logs, and check for
plugin/theme conflicts in a staging environment.

## Support Handoff

For unresolved plugin issues, contact `care@jeel.co` and include:

- the exact error message;
- WordPress, WooCommerce, PHP, and plugin versions;
- whether sandbox or production mode is active;
- relevant WooCommerce logs with credentials and customer data removed.
