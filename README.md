# WooCommerce Zibal Payment Gateway

This is a custom **WooCommerce payment gateway integration for Zibal**, a popular Iranian online payment service. This code allows you to add Zibal as a payment option on your WooCommerce checkout page, enabling your customers to pay for their orders securely through the Zibal payment portal.

## Why use this code?

* **Accept Payments via Zibal:** Integrate Zibal as a payment method for your WooCommerce store.
* **Secure Transactions:** Facilitates secure redirection to the Zibal gateway for payment processing.
* **Admin Configuration:** Provides an intuitive settings page within WooCommerce to configure your Zibal Merchant ID.
* **Automated Order Status Updates:** Automatically updates order statuses based on payment success or failure.
* **Error Handling:** Includes comprehensive error handling with user-friendly messages for various transaction outcomes.
* **Currency Conversion:** Automatically converts Toman/IRT to Rial for Zibal's API if your store's currency is set to Toman or IRR (though Zibal typically accepts Toman directly now, this code performs conversion for older setups or strict Rial APIs).

## Features

* Adds "Zibal" as a selectable payment gateway in WooCommerce settings.
* Allows configuration of gateway title, description, and Merchant ID.
* Handles the entire payment flow:
    * Initiates payment request to Zibal.
    * Redirects customer to Zibal payment page.
    * Manages the callback (return from Zibal).
    * Verifies payment with Zibal's API.
    * Completes WooCommerce order for successful payments.
    * Updates order status and adds notes for successful, cancelled, or failed payments.
* Translates Zibal API error codes into human-readable messages.

## Installation

This code functions as a custom WooCommerce payment gateway plugin.

1.  **Create Plugin Folder:** In your WordPress installation, navigate to `wp-content/plugins/` and create a new folder named `woocommerce-zibal-gateway`.
2.  **Create Plugin File:** Inside the `woocommerce-zibal-gateway` folder, create a file named `woocommerce-zibal-gateway.php`.
3.  **Paste Code:** Copy and paste the entire code provided into the `woocommerce-zibal-gateway.php` file.

    ```php
    <?php
    /**
     * Plugin Name: WooCommerce Zibal Payment Gateway
     * Description: Integrates Zibal as a payment gateway for WooCommerce.
     * Version: 1.0
     * Author: Your Name (Optional)
     * License: GPL-2.0-or-later
     * Text Domain: wc-zibal-gateway
     *
     * @package WooCommerce
     * @subpackage Zibal_Gateway
     * @author Masoud Aroodipoor
     */

    if ( ! defined( 'ABSPATH' ) ) {
        exit; // Exit if accessed directly.
    }

    /**
     * Adds the Zibal gateway class to WooCommerce.
     *
     * @param array $gateways Existing WooCommerce payment gateways.
     * @return array Modified list of gateways.
     */
    add_filter('woocommerce_payment_gateways', 'add_zibal_gateway_to_wc');
    function add_zibal_gateway_to_wc($gateways)
    {
        $gateways[] = 'WC_Zibal_Gateway';
        return $gateways;
    }

    /**
     * Initializes the Zibal payment gateway class.
     * Ensures WC_Payment_Gateway class is available before proceeding.
     */
    add_action('plugins_loaded', 'init_zibal_gateway');
    function init_zibal_gateway()
    {
        if (!class_exists('WC_Payment_Gateway')) {
            return;
        }

        class WC_Zibal_Gateway extends WC_Payment_Gateway
        {
            /**
             * @var string Merchant ID
             */
            public $merchant_id;

            /**
             * @var string Callback URL
             */
            public $callback_url;

            /**
             * Constructor for the gateway.
             */
            public function __construct()
            {
                $this->id = 'zibal';
                $this->method_title = 'Zibal'; // Displayed in admin
                $this->method_description = 'Secure payment through Zibal payment gateway.'; // Displayed in admin
                $this->has_fields = false; // No extra fields on checkout
                $this->icon = apply_filters('woocommerce_zibal_icon', plugin_dir_url(__FILE__) . 'zibal.png'); // Path to gateway icon

                // Initialize settings and form fields
                $this->init_form_fields();
                $this->init_settings();

                // Get values from settings
                $this->title = $this->get_option('title'); // Displayed on checkout page
                $this->description = $this->get_option('description'); // Displayed on checkout page
                $this->merchant_id = $this->get_option('merchant_id');
                // WooCommerce API callback URL: [yoursite.com/?wc-api=zibal_callback](https://yoursite.com/?wc-api=zibal_callback)
                $this->callback_url = WC()->api_request_url('zibal_callback');

                // Register hooks
                add_action('woocommerce_update_options_payment_gateways_' . $this->id, [$this, 'process_admin_options']);
                add_action('woocommerce_api_zibal_callback', [$this, 'handle_callback']); // Handles the return from Zibal
            }

            /**
             * Define gateway settings fields.
             */
            public function init_form_fields()
            {
                $this->form_fields = [
                    'enabled' => [
                        'title'   => __('Enable/Disable', 'wc-zibal-gateway'),
                        'type'    => 'checkbox',
                        'label'   => __('Enable Zibal Gateway', 'wc-zibal-gateway'),
                        'default' => 'yes',
                    ],
                    'title' => [
                        'title'       => __('Title', 'wc-zibal-gateway'),
                        'type'        => 'text',
                        'description' => __('This controls the title which the user sees during checkout.', 'wc-zibal-gateway'),
                        'default'     => __('Pay with Zibal Gateway', 'wc-zibal-gateway'),
                        'desc_tip'    => true,
                    ],
                    'description' => [
                        'title'       => __('Description', 'wc-zibal-gateway'),
                        'type'        => 'textarea',
                        'description' => __('This controls the description which the user sees during checkout.', 'wc-zibal-gateway'),
                        'default'     => __('Secure payment through Zibal gateway.', 'wc-zibal-gateway'),
                    ],
                    'merchant_id' => [
                        'title'       => __('Merchant ID', 'wc-zibal-gateway'),
                        'type'        => 'text',
                        'description' => __('Enter your Zibal Merchant ID. You can get this from your Zibal panel.', 'wc-zibal-gateway'),
                    ],
                ];
            }

            /**
             * Process the payment.
             *
             * @param int $order_id The ID of the order.
             * @return array An array of result and redirect URL.
             */
            public function process_payment($order_id)
            {
                $order = wc_get_order($order_id);
                $amount = (int) $order->get_total();

                // Convert Toman to Rial if currency is Toman/IRT (Zibal usually accepts Toman directly now)
                if (strtolower(get_woocommerce_currency()) === 'irt' || strtolower(get_woocommerce_currency()) === 'toman') {
                    $amount *= 10;
                }

                $data = [
                    'merchant'    => $this->merchant_id,
                    'amount'      => $amount,
                    'callbackUrl' => add_query_arg('order_id', $order_id, $this->callback_url),
                    'description' => 'Payment for order: ' . $order->get_order_number(),
                    'mobile'      => $order->get_billing_phone(),
                ];

                $response = $this->send_request_to_api('[https://gateway.zibal.ir/v1/request](https://gateway.zibal.ir/v1/request)', $data);

                if (isset($response['result']) && $response['result'] == 100) {
                    $track_id = $response['trackId'];
                    $redirect_url = '[https://gateway.zibal.ir/start/](https://gateway.zibal.ir/start/)' . $track_id;
                    
                    return [
                        'result'   => 'success',
                        'redirect' => $redirect_url,
                    ];
                } else {
                    $error_code = $response['result'] ?? 'unknown';
                    $error_message = $this->get_error_message($error_code);
                    wc_add_notice(__('Error connecting to payment gateway:', 'wc-zibal-gateway') . ' ' . $error_message, 'error');
                    return ['result' => 'failure'];
                }
            }

            /**
             * Handle the callback from Zibal gateway.
             */
            public function handle_callback()
            {
                if (empty($_GET['trackId']) || empty($_GET['success']) || empty($_GET['order_id'])) {
                    wc_add_notice(__('Incomplete return information from the gateway.', 'wc-zibal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }

                $order_id = absint($_GET['order_id']);
                $track_id = absint($_GET['trackId']);
                $success  = absint($_GET['success']);
                $status   = absint($_GET['status']); // 1: success, 2: already verified
                $order    = wc_get_order($order_id);
                
                if (!$order || !$order->get_id()) {
                    wc_add_notice(__('Order not found.', 'wc-zibal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }
                
                // If user cancelled the payment
                if ($success !== 1) {
                    $order->update_status('cancelled', __('Payment cancelled by user.', 'wc-zibal-gateway'));
                    wc_add_notice(__('Payment was unsuccessful or cancelled by you.', 'wc-zibal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }

                // If the transaction was already verified
                if ($status == 2) {
                    wc_add_notice(__('This transaction has already been verified.', 'wc-zibal-gateway'), 'success');
                    wp_redirect($this->get_return_url($order));
                    exit;
                }
                
                // Verify the transaction
                $data = [
                    'merchant' => $this->merchant_id,
                    'trackId'  => $track_id,
                ];
                
                $response = $this->send_request_to_api('[https://gateway.zibal.ir/v1/verify](https://gateway.zibal.ir/v1/verify)', $data);

                if (isset($response['result']) && $response['result'] == 100) {
                    $ref_number = $response['refNumber'];
                    $order->payment_complete($ref_number);
                    $order->add_order_note(__('Payment successful. Zibal Tracking ID:', 'wc-zibal-gateway') . ' ' . $ref_number);
                    wc_add_notice(__('Your payment was successful.', 'wc-zibal-gateway'), 'success');
                    wp_redirect($this->get_return_url($order));
                    exit;
                } else {
                    $error_code = $response['result'] ?? 'unknown';
                    $error_message = $this->get_error_message($error_code);
                    $order->update_status('failed', __('Payment failed:', 'wc-zibal-gateway') . ' ' . $error_message);
                    wc_add_notice(__('Error in transaction verification:', 'wc-zibal-gateway') . ' ' . $error_message, 'error');
                }

                wp_redirect(wc_get_checkout_url());
                exit;
            }

            /**
             * Sends a request to the Zibal API.
             *
             * @param string $url  API endpoint URL.
             * @param array  $data Data to send in the request body.
             * @return array Decoded JSON response from the API.
             */
            private function send_request_to_api($url, $data) {
                $response = wp_remote_post($url, [
                    'method'  => 'POST',
                    'headers' => ['Content-Type' => 'application/json'],
                    'body'    => json_encode($data),
                    'timeout' => 30, // seconds
                ]);

                if (is_wp_error($response)) {
                    error_log('Zibal API Connection Error: ' . $response->get_error_message());
                    return ['result' => -999]; // Custom code for connection error
                }
                
                return json_decode(wp_remote_retrieve_body($response), true);
            }

            /**
             * Translates Zibal error codes into human-readable messages.
             *
             * @param int|string $code The Zibal error code.
             * @return string Translated error message.
             */
            private function get_error_message($code) {
                $errors = [
                    '100'     => __('Transaction was successful.', 'wc-zibal-gateway'),
                    '102'     => __('Merchant ID is not active.', 'wc-zibal-gateway'),
                    '103'     => __('Invalid Merchant ID.', 'wc-zibal-gateway'),
                    '104'     => __('Amount is more than the allowed limit.', 'wc-zibal-gateway'),
                    '105'     => __('Amount is less than the allowed minimum.', 'wc-zibal-gateway'),
                    '106'     => __('Invalid callback URL.', 'wc-zibal-gateway'),
                    '113'     => __('Transaction amount and verified amount are not equal.', 'wc-zibal-gateway'),
                    '201'     => __('Transaction has already been verified.', 'wc-zibal-gateway'),
                    '202'     => __('Order not paid or was unsuccessful.', 'wc-zibal-gateway'),
                    '203'     => __('Invalid Track ID.', 'wc-zibal-gateway'),
                    '-999'    => __('Error connecting to the server.', 'wc-zibal-gateway'),
                    'unknown' => __('An unknown error occurred.', 'wc-zibal-gateway'),
                ];

                return $errors[$code] ?? sprintf(__('Undefined error with code: %s', 'wc-zibal-gateway'), $code);
            }
        }
    }
    ```

4.  **Upload Icon (Optional but Recommended):** Place your Zibal payment gateway icon (e.g., `zibal.png`) inside the `woocommerce-zibal-gateway` folder, next to `woocommerce-zibal-gateway.php`. This icon will appear next to the payment gateway name on the checkout page.
5.  **Activate Plugin:** Go to your WordPress admin dashboard, navigate to **"Plugins"**, and **activate** the "WooCommerce Zibal Payment Gateway" plugin.
6.  **Configure Gateway:** Go to **WooCommerce > Settings > Payments**. You should now see "Zibal" listed. Click on "Manage" (یا "Set up") to configure your Merchant ID and other settings.

## Important Considerations

* **Merchant ID:** You MUST obtain a valid Merchant ID from Zibal to use this gateway.
* **Currency:** The code includes logic to convert Toman/IRT to Rial (`$amount *= 10`). While Zibal's API often accepts Toman directly, this conversion ensures compatibility if your WooCommerce currency is set to Toman or IRR and Zibal expects Rial for specific operations or older API versions. **Verify Zibal's current API requirements for currency.**
* **SSL Certificate:** For security, ensure your website has a valid SSL certificate (HTTPS) installed. Payment gateways require secure connections.
* **Error Logging:** The `error_log` calls are useful for debugging connection issues. Check your server's PHP error logs if you encounter problems.
* **Security:** This is a basic implementation. For production environments, always consider additional security best practices, such as IP whitelisting if supported by Zibal, and regular security audits.
* **Updates:** Custom gateway code needs to be maintained. Ensure you stay updated with any changes in Zibal's API or WooCommerce's payment gateway standards.

## Contributing

Contributions are welcome! If you have suggestions or improvements for this code, feel free to open a "Pull Request" or report an "Issue."

## License

This project is licensed under the GPL-2.0-or-later License.

---


# درگاه پرداخت زیبال برای ووکامرس

این کد یکپارچه‌سازی سفارشی **درگاه پرداخت زیبال برای ووکامرس** است. زیبال یک سرویس پرداخت آنلاین محبوب ایرانی است. این کد به شما امکان می‌دهد زیبال را به عنوان یک گزینه پرداخت در صفحه تسویه‌حساب ووکامرس خود اضافه کنید و مشتریان خود را قادر می‌سازد تا سفارشات خود را به طور امن از طریق پورتال پرداخت زیبال پرداخت کنند.

## چرا از این کد استفاده کنیم؟

* **پذیرش پرداخت از طریق زیبال:** زیبال را به عنوان یک روش پرداخت برای فروشگاه ووکامرس خود ادغام کنید.
* **تراکنش‌های امن:** انتقال امن به درگاه زیبال برای پردازش پرداخت را تسهیل می‌کند.
* **پیکربندی مدیر:** یک صفحه تنظیمات بصری در ووکامرس برای پیکربندی مرچنت کد زیبال شما فراهم می‌کند.
* **به‌روزرسانی خودکار وضعیت سفارش:** وضعیت سفارش را به طور خودکار بر اساس موفقیت یا عدم موفقیت پرداخت به‌روزرسانی می‌کند.
* **مدیریت خطا:** شامل مدیریت خطای جامع با پیام‌های کاربرپسند برای نتایج مختلف تراکنش‌ها است.
* **تبدیل ارز:** اگر ارز فروشگاه شما تومان یا IRR باشد، به طور خودکار تومان/IRT را به ریال برای API زیبال تبدیل می‌کند (اگرچه زیبال معمولاً اکنون تومان را مستقیماً می‌پذیرد، این کد برای تنظیمات قدیمی‌تر یا APIهای سختگیرانه ریالی تبدیل را انجام می‌دهد).

## قابلیت‌ها

* "زیبال" را به عنوان یک درگاه پرداخت قابل انتخاب در تنظیمات ووکامرس اضافه می‌کند.
* امکان پیکربندی عنوان درگاه، توضیحات و مرچنت کد را فراهم می‌کند.
* تمام جریان پرداخت را مدیریت می‌کند:
    * آغاز درخواست پرداخت به زیبال.
    * هدایت مشتری به صفحه پرداخت زیبال.
    * مدیریت بازگشت از زیبال (callback).
    * تأیید پرداخت با API زیبال.
    * تکمیل سفارش ووکامرس برای پرداخت‌های موفق.
    * به‌روزرسانی وضعیت سفارش و افزودن یادداشت‌ها برای پرداخت‌های موفق، لغو شده یا ناموفق.
* کدهای خطای API زیبال را به پیام‌های قابل فهم برای انسان ترجمه می‌کند.

## نصب

این کد به عنوان یک افزونه درگاه پرداخت سفارشی ووکامرس عمل می‌کند.

1.  **ایجاد پوشه افزونه:** در نصب وردپرس خود، به مسیر `wp-content/plugins/` بروید و یک پوشه جدید به نام `woocommerce-zibal-gateway` ایجاد کنید.
2.  **ایجاد فایل افزونه:** در داخل پوشه `woocommerce-zibal-gateway`، یک فایل به نام `woocommerce-zibal-gateway.php` ایجاد کنید.
3.  **جایگذاری کد:** کل کد ارائه شده را کپی کرده و در فایل `woocommerce-zibal-gateway.php` جایگذاری کنید.

    ```php
    <?php
    /**
     * Plugin Name: WooCommerce Zibal Payment Gateway
     * Description: Integrates Zibal as a payment gateway for WooCommerce.
     * Version: 1.0
     * Author: Your Name (Optional)
     * License: GPL-2.0-or-later
     * Text Domain: wc-zibal-gateway
     *
     * @package WooCommerce
     * @subpackage Zibal_Gateway
     * @author Masoud Aroodipoor
     */

    if ( ! defined( 'ABSPATH' ) ) {
        exit; // Exit if accessed directly.
    }

    /**
     * Adds the Zibal gateway class to WooCommerce.
     *
     * @param array $gateways Existing WooCommerce payment gateways.
     * @return array Modified list of gateways.
     */
    add_filter('woocommerce_payment_gateways', 'add_zibal_gateway_to_wc');
    function add_zibal_gateway_to_wc($gateways)
    {
        $gateways[] = 'WC_Zibal_Gateway';
        return $gateways;
    }

    /**
     * Initializes the Zibal payment gateway class.
     * Ensures WC_Payment_Gateway class is available before proceeding.
     */
    add_action('plugins_loaded', 'init_zibal_gateway');
    function init_zibal_gateway()
    {
        if (!class_exists('WC_Payment_Gateway')) {
            return;
        }

        class WC_Zibal_Gateway extends WC_Payment_Gateway
        {
            /**
             * @var string کد مرچنت
             */
            public $merchant_id;

            /**
             * @var string آدرس بازگشت از درگاه
             */
            public $callback_url;

            /**
             * سازنده کلاس
             */
            public function __construct()
            {
                $this->id = 'zibal';
                $this->method_title = 'زیبال'; // نمایش در مدیریت
                $this->method_description = 'پرداخت امن از طریق درگاه پرداخت زیبال'; // نمایش در مدیریت
                $this->has_fields = false; // بدون فیلدهای اضافی در تسویه‌حساب
                $this->icon = apply_filters('woocommerce_zibal_icon', plugin_dir_url(__FILE__) . 'zibal.png'); // مسیر آیکون درگاه

                // مقداردهی اولیه تنظیمات و فیلدهای فرم
                $this->init_form_fields();
                $this->init_settings();

                // دریافت مقادیر از تنظیمات
                $this->title = $this->get_option('title'); // نمایش در صفحه تسویه‌حساب
                $this->description = $this->get_option('description'); // نمایش در صفحه تسویه‌حساب
                $this->merchant_id = $this->get_option('merchant_id');
                // آدرس بازگشت API ووکامرس: yoursite.com/?wc-api=zibal_callback
                $this->callback_url = WC()->api_request_url('zibal_callback');

                // ثبت هوک‌ها
                add_action('woocommerce_update_options_payment_gateways_' . $this->id, [$this, 'process_admin_options']);
                add_action('woocommerce_api_zibal_callback', [$this, 'handle_callback']); // بازگشت از زیبال را مدیریت می‌کند
            }

            /**
             * تعریف فیلدهای تنظیمات درگاه
             */
            public function init_form_fields()
            {
                $this->form_fields = [
                    'enabled' => [
                        'title'   => __('فعال‌سازی/غیرفعال‌سازی', 'wc-zibal-gateway'),
                        'type'    => 'checkbox',
                        'label'   => __('فعال کردن درگاه زیبال', 'wc-zibal-gateway'),
                        'default' => 'yes',
                    ],
                    'title' => [
                        'title'       => __('عنوان', 'wc-zibal-gateway'),
                        'type'        => 'text',
                        'description' => __('این فیلد عنوانی است که کاربر در صفحه تسویه حساب مشاهده می‌کند.', 'wc-zibal-gateway'),
                        'default'     => __('پرداخت با درگاه زیبال', 'wc-zibal-gateway'),
                        'desc_tip'    => true,
                    ],
                    'description' => [
                        'title'       => __('توضیحات', 'wc-zibal-gateway'),
                        'type'        => 'textarea',
                        'description' => __('این فیلد توضیحاتی است که کاربر در صفحه تسویه حساب مشاهده می‌کند.', 'wc-zibal-gateway'),
                        'default'     => __('پرداخت امن از طریق درگاه زیبال.', 'wc-zibal-gateway'),
                    ],
                    'merchant_id' => [
                        'title'       => __('مرچنت کد (Merchant)', 'wc-zibal-gateway'),
                        'type'        => 'text',
                        'description' => __('کد مرچنت خود را از پنل زیبال وارد کنید.', 'wc-zibal-gateway'),
                    ],
                ];
            }

            /**
             * پردازش پرداخت
             *
             * @param int $order_id شناسه سفارش.
             * @return array آرایه‌ای شامل نتیجه و URL هدایت.
             */
            public function process_payment($order_id)
            {
                $order = wc_get_order($order_id);
                $amount = (int) $order->get_total();

                // تبدیل تومان به ریال در صورت نیاز (زیبال معمولاً اکنون تومان را مستقیماً می‌پذیرد)
                if (strtolower(get_woocommerce_currency()) === 'irt' || strtolower(get_woocommerce_currency()) === 'toman') {
                    $amount *= 10;
                }

                $data = [
                    'merchant'    => $this->merchant_id,
                    'amount'      => $amount,
                    'callbackUrl' => add_query_arg('order_id', $order_id, $this->callback_url),
                    'description' => 'پرداخت سفارش شماره: ' . $order->get_order_number(),
                    'mobile'      => $order->get_billing_phone(),
                ];

                $response = $this->send_request_to_api('https://gateway.zibal.ir/v1/request', $data);

                if (isset($response['result']) && $response['result'] == 100) {
                    $track_id = $response['trackId'];
                    $redirect_url = 'https://gateway.zibal.ir/start/' . $track_id;
                    
                    return [
                        'result'   => 'success',
                        'redirect' => $redirect_url,
                    ];
                } else {
                    $error_code = $response['result'] ?? 'unknown';
                    $error_message = $this->get_error_message($error_code);
                    wc_add_notice(__('خطا در اتصال به درگاه پرداخت:', 'wc-zibal-gateway') . ' ' . $error_message, 'error');
                    return ['result' => 'failure'];
                }
            }

            /**
             * مدیریت بازگشت از درگاه زیبال
             */
            public function handle_callback()
            {
                if (empty($_GET['trackId']) || empty($_GET['success']) || empty($_GET['order_id'])) {
                    wc_add_notice(__('اطلاعات بازگشتی از درگاه ناقص است.', 'wc-zibal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }

                $order_id = absint($_GET['order_id']);
                $track_id = absint($_GET['trackId']);
                $success  = absint($_GET['success']);
                $status   = absint($_GET['status']); // 1: success, 2: already verified
                $order    = wc_get_order($order_id);
                
                if (!$order || !$order->get_id()) {
                    wc_add_notice(__('سفارش یافت نشد.', 'wc-zibal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }
                
                // اگر کاربر پرداخت را لغو کرده باشد
                if ($success !== 1) {
                    $order->update_status('cancelled', __('پرداخت توسط کاربر لغو شد.', 'wc-zibal-gateway'));
                    wc_add_notice(__('پرداخت ناموفق بود یا توسط شما لغو شد.', 'wc-zibal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }

                // اگر تراکنش قبلاً وریفای شده است
                if ($status == 2) {
                    wc_add_notice(__('این تراکنش قبلاً تایید شده است.', 'wc-zibal-gateway'), 'success');
                    wp_redirect($this->get_return_url($order));
                    exit;
                }
                
                // تایید تراکنش
                $data = [
                    'merchant' => $this->merchant_id,
                    'trackId'  => $track_id,
                ];
                
                $response = $this->send_request_to_api('https://gateway.zibal.ir/v1/verify', $data);

                if (isset($response['result']) && $response['result'] == 100) {
                    $ref_number = $response['refNumber'];
                    $order->payment_complete($ref_number);
                    $order->add_order_note(__('پرداخت موفق. شماره پیگیری زیبال:', 'wc-zibal-gateway') . ' ' . $ref_number);
                    wc_add_notice(__('پرداخت شما با موفقیت انجام شد.', 'wc-zibal-gateway'), 'success');
                    wp_redirect($this->get_return_url($order));
                    exit;
                } else {
                    $error_code = $response['result'] ?? 'unknown';
                    $error_message = $this->get_error_message($error_code);
                    $order->update_status('failed', __('پرداخت ناموفق:', 'wc-zibal-gateway') . ' ' . $error_message);
                    wc_add_notice(__('خطا در تایید تراکنش:', 'wc-zibal-gateway') . ' ' . $error_message, 'error');
                }

                wp_redirect(wc_get_checkout_url());
                exit;
            }

            /**
             * ارسال درخواست به API زیبال
             *
             * @param string $url  آدرس نقطه پایانی API.
             * @param array  $data داده برای ارسال در بدنه درخواست.
             * @return array پاسخ JSON رمزگشایی شده از API.
             */
            private function send_request_to_api($url, $data) {
                $response = wp_remote_post($url, [
                    'method'  => 'POST',
                    'headers' => ['Content-Type' => 'application/json'],
                    'body'    => json_encode($data),
                    'timeout' => 30, // ثانیه
                ]);

                if (is_wp_error($response)) {
                    error_log('Zibal API Connection Error: ' . $response->get_error_message());
                    return ['result' => -999]; // کد سفارشی برای خطای اتصال
                }
                
                return json_decode(wp_remote_retrieve_body($response), true);
            }

            /**
             * ترجمه کدهای خطای زیبال به پیام‌های قابل فهم برای انسان
             *
             * @param int|string $code کد خطای زیبال.
             * @return string پیام خطای ترجمه شده.
             */
            private function get_error_message($code) {
                $errors = [
                    '100'     => __('تراکنش با موفقیت انجام شد.', 'wc-zibal-gateway'),
                    '102'     => __('مرچنت کد فعال نیست.', 'wc-zibal-gateway'),
                    '103'     => __('مرچنت کد نامعتبر است.', 'wc-zibal-gateway'),
                    '104'     => __('مبلغ بیشتر از سقف مجاز است.', 'wc-zibal-gateway'),
                    '105'     => __('مبلغ کمتر از حداقل مجاز است.', 'wc-zibal-gateway'),
                    '106'     => __('آدرس بازگشتی (callbackUrl) نامعتبر است.', 'wc-zibal-gateway'),
                    '113'     => __('مبلغ تراکنش و مبلغ وریفای شده یکسان نیستند.', 'wc-zibal-gateway'),
                    '201'     => __('تراکنش قبلاً وریفای شده است.', 'wc-zibal-gateway'),
                    '202'     => __('سفارش پرداخت نشده یا ناموفق بوده است.', 'wc-zibal-gateway'),
                    '203'     => __('کد رهگیری (trackId) نامعتبر است.', 'wc-zibal-gateway'),
                    '-999'    => __('خطا در برقراری ارتباط با سرور.', 'wc-zibal-gateway'),
                    'unknown' => __('خطای نامشخص رخ داده است.', 'wc-zibal-gateway'),
                ];

                return $errors[$code] ?? sprintf(__('خطای تعریف نشده با کد: %s', 'wc-zibal-gateway'), $code);
            }
        }
    }
    ```

4.  **آپلود آیکون (اختیاری اما توصیه می‌شود):** آیکون درگاه پرداخت زیبال خود (مثلاً `zibal.png`) را در داخل پوشه `woocommerce-zibal-gateway`، کنار `woocommerce-zibal-gateway.php` قرار دهید. این آیکون در کنار نام درگاه پرداخت در صفحه تسویه‌حساب ظاهر می‌شود.
5.  **فعال‌سازی افزونه:** وارد پنل مدیریت وردپرس خود شوید، به بخش **"افزونه‌ها"** بروید و افزونه **"درگاه پرداخت زیبال برای ووکامرس"** را **فعال کنید**.
6.  **پیکربندی درگاه:** به **ووکامرس > تنظیمات > پرداخت‌ها** بروید. اکنون باید "زیبال" را در لیست مشاهده کنید. برای پیکربندی مرچنت کد و سایر تنظیمات، روی "مدیریت" (یا "راه‌اندازی") کلیک کنید.

## ملاحظات مهم

* **مرچنت کد:** برای استفاده از این درگاه **باید** یک مرچنت کد معتبر از زیبال دریافت کنید.
* **واحد پول:** این کد شامل منطق تبدیل تومان/IRT به ریال (`$amount *= 10`) است. در حالی که API زیبال اغلب تومان را مستقیماً می‌پذیرد، این تبدیل سازگاری را تضمین می‌کند اگر ارز ووکامرس شما روی تومان یا IRR تنظیم شده باشد و زیبال برای عملیات خاص یا نسخه‌های قدیمی‌تر API انتظار ریال را داشته باشد. **الزامات فعلی API زیبال برای واحد پول را بررسی کنید.**
* **گواهی SSL:** برای امنیت، اطمینان حاصل کنید که وب‌سایت شما دارای گواهی SSL معتبر (HTTPS) است. درگاه‌های پرداخت نیاز به اتصالات امن دارند.
* **لاگ خطا:** فراخوانی‌های `error_log` برای اشکال‌زدایی مشکلات اتصال مفید هستند. در صورت بروز مشکل، لاگ خطاهای PHP سرور خود را بررسی کنید.
* **امنیت:** این یک پیاده‌سازی پایه است. برای محیط‌های تولید، همیشه بهترین روش‌های امنیتی اضافی را در نظر بگیرید، مانند لیست سفید IP (در صورت پشتیبانی توسط زیبال) و ممیزی‌های امنیتی منظم.
* **به‌روزرسانی‌ها:** کد درگاه سفارشی نیاز به نگهداری دارد. اطمینان حاصل کنید که با هرگونه تغییر در API زیبال یا استانداردهای درگاه پرداخت ووکامرس به‌روز می‌مانید.

## مشارکت (Contributing)

مشارکت شما خوشایند است! اگر پیشنهاد یا بهبودهایی برای این کد دارید، می‌توانید یک "Pull Request" ایجاد کنید یا "Issue" جدیدی را گزارش دهید.

## مجوز (License)

این پروژه تحت مجوز GPL-2.0-or-later منتشر شده است.
