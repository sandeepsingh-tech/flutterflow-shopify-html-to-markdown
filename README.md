### Render Rich Shopify Product Descriptions in FlutterFlow

Shopify’s Storefront API returns product descriptions as raw HTML (`descriptionHtml`), which often includes messy tags, empty paragraphs, inline styling, and vendor formatting artifacts. Because FlutterFlow's native **Markdown** widget cannot render raw HTML directly, converting `descriptionHtml` into structured Markdown ensures a polished mobile shopping experience.

#### Why Proper Product Description Formatting Matters

* **Higher Buyer Trust & Conversion Rates:** Cleanly structured headings, bold highlights, and bulleted specifications make products look professional and trustworthy—eliminating broken HTML tags (`<p>`, `&nbsp;`, `**`) that cause shoppers to abandon checkout.
* **Effortless Mobile Scannability:** Mobile shoppers rarely read walls of text. Separating descriptions into clear sections (*Key Features*, *Dimensions*, *FAQ*) lets customers find critical details like size, weight, and compatibility in seconds.
* **Fewer Customer Support Queries & Returns:** Clearly highlighted dimensions, weight specs, and FAQs set accurate expectations before purchase, directly reducing "item not as expected" returns and repetitive pre-sale support tickets.
* **Zero Performance Overhead:** Unlike embedding heavy `WebView` components on every product page, a lightweight Dart parser paired with FlutterFlow’s native Markdown widget renders instantaneously and adapts seamlessly to your app’s typography and dark/light theme.

---

### Step 1: Configure the Shopify Storefront GraphQL API Call

1. In FlutterFlow, go to **API Calls** in the left navigation menu and create a new API call named `getProductDetails` (or add it to your existing Shopify API Group).
2. Set the **Method Type** to `POST` and enter your Storefront GraphQL endpoint:
`[https://YOUR-STORE-NAME.myshopify.com/api/2026-01/graphql.json](https://YOUR-STORE-NAME.myshopify.com/api/2026-01/graphql.json)`
3. Add your required headers:
* `Content-Type`: `application/json`
* `X-Shopify-Storefront-Access-Token`: `YOUR_STOREFRONT_ACCESS_TOKEN`


4. Under **Variables**, create a variable named `productId` of type `String`.
5. Go to the **Body** tab, select **JSON**, and paste the following GraphQL query:

```json
{
  "query": "query getProductDetails($productId: ID!) { product(id: $productId) { id handle title description descriptionHtml options { name values } images(first: 20) { edges { node { url altText } } } variants(first: 50) { edges { node { id title availableForSale price { amount currencyCode } selectedOptions { name value } image { url altText } } } } } recommendations: productRecommendations(productId: $productId, intent: RELATED) { id title featuredImage { url altText } priceRange { minVariantPrice { amount currencyCode } } } }",
  "variables": {
    "productId": "<productId>"
  }
}

```

6. Click **Response & Test**, enter a valid Shopify Product GID (e.g., `gid://shopify/Product/1234567890`), and run a test call.
7. In the **Recommended** or **JSON Paths** tab, create a JSON path for the HTML description:
* **Name:** `descriptionHtml`
* **JSON Path:** `$.data.product.descriptionHtml`



---

### Step 2: Create the HTML-to-Markdown Custom Function

1. Navigate to **Custom Code** -> **Custom Functions** and click **+ Create**.
2. Name the function `shopifyHtmlToMarkdown` (or `shopifyHtmlToMarkdownCopy`).
3. Configure the **Function Settings** on the right panel:
* **Return Type:** `String` (check **Nullable**)
* **Arguments:** Add one argument named `htmlText` of type `String` (check **Nullable**)


4. Paste the following code inside the function body (between the `MODIFY CODE` markers):

```dart
  /// MODIFY CODE ONLY BELOW THIS LINE

  if (htmlText == null || htmlText.trim().isEmpty) {
    return 'No description available.';
  }

  String text = htmlText;

  // 1. Remove empty or whitespace-only tags
  text = text.replaceAll(
      RegExp(r'<(strong|b|em|i|u|h\d|p|li|span)>\s*<\/\1>',
          caseSensitive: false),
      '');
  text = text.replaceAll(
      RegExp(r'<li[^>]*>(\s|&nbsp;|<br\s*\/?>)*<\/li>', caseSensitive: false),
      '');
  text = text.replaceAll(
      RegExp(r'<p[^>]*>(\s|&nbsp;|<br\s*\/?>)*<\/p>', caseSensitive: false),
      '');

  // 2. Normalize standard HTML headings (h1 - h6)
  text = text.replaceAllMapped(
    RegExp(r'<h[1-6][^>]*>(.*?)<\/h[1-6]>', dotAll: true),
    (match) {
      final content =
          match.group(1)?.replaceAll(RegExp(r'<[^>]*>'), '').trim() ?? '';
      return content.isNotEmpty ? '\n\n**$content**\n\n' : '';
    },
  );

  // 3. Format major section headings
  final sectionTitles = [
    'Description',
    'Key Features',
    'Ideal For',
    'Specifications',
    'Dimension',
    'Dimensions',
    'FAQ',
    'Frequently Asked Questions',
  ];

  for (final title in sectionTitles) {
    text = text.replaceAll(
      RegExp(
          '(<strong[^>]*>|<b[^>]*>)?' +
              RegExp.escape(title) +
              r'\s*(:|-|:-)(<\/strong>|<\/b>)?',
          caseSensitive: false),
      '\n\n**$title**\n\n',
    );
  }

  // 4. Convert <li> items to markdown bullets (only non-empty items)
  text = text.replaceAllMapped(
    RegExp(r'<li[^>]*>(.*?)<\/li>', dotAll: true),
    (match) {
      final cleanItem =
          match.group(1)?.replaceAll(RegExp(r'<[^>]*>'), '').trim() ?? '';
      return cleanItem.isNotEmpty ? '\n* $cleanItem' : '';
    },
  );

  // 5. Structure Dimensions cleanly
  final dimensionKeys = [
    'Volu. Weight',
    'Vol. Weight',
    'Product Weight',
    'Ship Weight',
    'Length',
    'Breadth',
    'Width',
    'Height',
    'Thickness',
  ];

  for (final key in dimensionKeys) {
    text = text.replaceAllMapped(
      RegExp(
          r'(?:<br\s*\/?>|\n|\s{2,})?' +
              RegExp.escape(key) +
              r'\s*(?:\([^)]*\))?\s*(?::-|:)\s*',
          caseSensitive: false),
      (match) => '\n* **$key:** ',
    );
  }

  // 6. Structure FAQ (Q1:, A:)
  text = text.replaceAllMapped(
    RegExp(r'(?:<br\s*\/?>|\n|\s{2,})?(Q\d+:?|Question\s*\d*:?)\s*',
        caseSensitive: false),
    (match) => '\n\n**${match.group(1)?.toUpperCase()}** ',
  );

  text = text.replaceAllMapped(
    RegExp(r'(?:<br\s*\/?>|\s{2,})?(A:|Answer:)\s*', caseSensitive: false),
    (match) => '\n*Answer:* ',
  );

  // 7. Convert remaining bold/italic tags
  text = text.replaceAllMapped(
    RegExp(r'<(?:strong|b)[^>]*>(.*?)<\/(?:strong|b)>', dotAll: true),
    (match) {
      final inner = match.group(1)?.trim() ?? '';
      return inner.isNotEmpty ? '**$inner**' : '';
    },
  );

  text = text.replaceAllMapped(
    RegExp(r'<(?:em|i)[^>]*>(.*?)<\/(?:em|i)>', dotAll: true),
    (match) {
      final inner = match.group(1)?.trim() ?? '';
      return inner.isNotEmpty ? '*$inner*' : '';
    },
  );

  // 8. Clean line breaks and strip all remaining HTML tags
  text = text.replaceAll(RegExp(r'<br\s*\/?>'), '\n');
  text = text.replaceAll(RegExp(r'<\/p>'), '\n\n');
  text = text.replaceAll(RegExp(r'<[^>]*>'), '');

  // 9. Decode HTML entities
  text = text
      .replaceAll('&amp;', '&')
      .replaceAll('&nbsp;', ' ')
      .replaceAll('&quot;', '"')
      .replaceAll('&#39;', "'")
      .replaceAll('&apos;', "'");

  // 10. Eliminate artifacts (_**, **, solitary symbols, orphan bullets)
  text = text.replaceAll(RegExp(r'[_*]{2,}'), '');
  text = text.replaceAll(RegExp(r'^\s*[_*\-\.]+\s*$', multiLine: true), '');
  text = text.replaceAll(RegExp(r'^\s*[\*\-]\s*$', multiLine: true), '');

  // 11. Normalize whitespace and clean excessive newlines
  text = text.replaceAll(RegExp(r'[ \t]+'), ' ');
  text = text.replaceAll(RegExp(r'\n{3,}'), '\n\n');

  return text.trim().isEmpty ? 'No description available.' : text.trim();

  /// MODIFY CODE ONLY ABOVE THIS LINE

```

5. Click **Save Function** and verify there are no compilation errors.

---

### Step 3: Bind the Function to the Markdown Widget

1. Open your **Product Details** page in FlutterFlow.
2. Ensure the page (or parent container) has a **Backend Query** calling `getProductDetails` with the page's `productId` parameter.
3. Drag and drop a **Markdown** widget (found under **Base Elements** in the Widget Palette) into your product description section.
4. Select the **Markdown** widget and open the **Properties Panel** on the right:
* Locate the **Data** / **Markdown Data** field and click the **Set from Variable** (orange cylinder) icon.
* Select **Custom Functions** -> **shopifyHtmlToMarkdown**.
* For the `htmlText` argument, select your `getProductDetails` API response -> **JSON Body** -> **Predefined Path** -> `descriptionHtml`.
* Set a **Default Variable Value** such as `Loading description...`.


5. *(Optional)* Customize the **Text Style**, **Heading Styles**, and toggle **Selectable** if you want shoppers to be able to copy product specifications.

---

### How the Parser Cleans Supplier HTML

| Stage | What It Fixes |
| --- | --- |
| **Empty Tag Stripping** | Removes blank `<p></p>`, `<li></li>`, and `<br>` chains often left behind by rich-text editors. |
| **Section & Spec Formatting** | Automatically detects labels like `Key Features:`, `Specifications:`, `Dimensions:`, `Length:`, and `Product Weight:` and converts them into bold headings or clean bullet points. |
| **FAQ Structuring** | Formats inline `Q1:` / `A:` patterns into spaced, readable question-and-answer blocks. |
| **Artifact Cleanup** | Strips broken formatting remnants (such as stray `**`, `_**`, or orphan bullets) that commonly appear in imported catalogs. |
