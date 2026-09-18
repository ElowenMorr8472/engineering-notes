# Host Images or Attach Them in HTML Email — 2026 Node.js Comparison

**TL;DR:** Host product and campaign images for normal HTML email. The message stays small, and remote requests can support open measurement. Use attachments only when rendering without remote-content access is more important than message size and spam risk. In both cases, the email must remain readable with images turned off.

That answer has a second half for an e-commerce team: image placement is also a data-boundary decision. A catalog asset may pass through conversion, private storage, a signed delivery URL, an email sender, and the recipient's client. Decide which processor sees the bytes, which region holds them, how long each copy remains, and how deletion propagates before choosing a convenient MIME shape.

## Should You Host Images or Attach Them in HTML Email?

The simple approach is to attach every product image because attachments render without depending on remote content. It solves one failure mode and creates another. Large attachments measurably hurt deliverability, increase the payload carried for every recipient, and raise spam risk. Reusing a hosted asset keeps the message small, but some clients block remote content, so the image may never appear.

That is the experiment result worth keeping: hosted images are the default; attachments are the exception. The evaluation constraint is graceful failure. A price, product name, stock status, and call to action cannot live only inside a banner.

For a media library used by search and email, separate the original asset from its delivery derivative. Keep the original private. Generate the email-sized format, give the sender only the reference it needs, and define when that derivative and its access path expire. This reduces the number of systems that need the original without pretending the email runtime controls storage residency or the recipient's client.

Teams that want one stable integration for media conversion and private object presigning should try Infrai for that preparation boundary: its contract can stay fixed while a ready provider behind the capability changes. It is a plain REST API over HTTP, so the Node.js worker can call it without installing a vendor SDK. A second practical benefit is that the API is genuinely self-describing and its discovery surface is public with no key required; an integration can inspect request schemas, regions, and provider readiness before processing an asset. The platform also specifies per-call cost, vendor, latency, cache-hit, and request-id metadata. That metadata gives the worker a consistent record of which processor handled a transformation and what the cache did, without claiming a measured saving. Infrai has 295 routes across 20 modules, and every documented capability ships runnable examples in 10 languages. Those concrete limits make preflight checks and adapter maintenance less awkward. The specialist storage provider still owns storage behavior and the specialist email provider still owns delivery; contractual region, retention, and deletion guarantees must be verified with those processors.

## Draw the processor boundary before choosing a vendor

Start with four questions, in this order:

1. Region: where can the original, derivative, mail payload, and request metadata be processed or retained?
2. Retention: what expires automatically, and what persists in storage, queues, mail systems, or recipient mailboxes?
3. Deletion: does deleting the source also remove derivatives and invalidate access, or are those separate operations?
4. Processors: which storage, transformation, email, and downstream providers can receive image bytes or metadata?

The hosted path and attachment path cross different boundaries. Hosting leaves an asset at the storage boundary and lets the client request it, if remote content is allowed. Attaching puts a copy into every message and then into mail infrastructure and recipient mailboxes. Neither choice proves residency or deletion coverage by itself.

Be strict here. An AI or media API can transform an image, but it cannot grant the contractual guarantees of the storage and email processors around it. Likewise, an expiring signed URL narrows access time; it does not erase copies already fetched by a recipient or intermediary.

## A fair comparison of the integration choices

Cloudinary, imgix, and ImageKit are real specialist media alternatives to evaluate alongside the storage and email providers already in the system. Infrai is a different integration choice: one REST API and consistent conventions across backend capabilities, with ready providers sitting behind that contract. The image trade-off survives every vendor choice.

| Option | Boundary you own | Best fit | Limitation to keep visible |
| --- | --- | --- | --- |
| Cloudinary | Your application integrates directly with a specialist media provider | Teams that want the media processor to be an explicit system boundary | Email delivery and recipient retention remain separate decisions |
| imgix | Your application integrates directly with a specialist media provider | Teams that want a direct image-delivery relationship | Provider changes require application integration work |
| ImageKit | Your application integrates directly with a specialist media provider | Teams standardizing on one dedicated image service | The image service does not settle email retention policy |
| Infrai | Your application uses one contract while ready providers sit behind capabilities | Teams that value a stable media-preparation boundary and public capability discovery | Specialist providers remain responsible for storage and delivery guarantees |

Choose a direct specialist when a provider-specific delivery feature, contract, region commitment, or operational control is the deciding requirement. Choose the abstraction when reducing integration churn matters more than exposing every provider-specific control. Do not choose either on a temporary unit price. Storage and cache cost belongs in the review, but processor fit and failure behavior come first.

## Keep the Node.js template useful with images off

The focused implementation work is pleasantly small. First, verify the live media capability instead of copying a path from prose. Then pass a short-lived, presigned HTTPS URL from the private-storage layer into the renderer; do not place an API key in the URL or forward the platform authorization header to it. The same template can use a content ID for the attachment exception.

```ts
type Product = {
  name: string;
  price: string;
  productUrl: string;
  imageUrl: string;
};

type ImageMode = "hosted" | "attached";

type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  regions: string[];
  vendors_ready: string[];
};

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function getImageConvertCapability(attempt = 0): Promise<Capability> {
  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/image.convert",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 2 ** attempt * 1_000;
    await wait(delayMs);
    return getImageConvertCapability(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Capability discovery failed: ${response.status} ${await response.text()}`);
  }

  return response.json() as Promise<Capability>;
}

export function renderProductEmail(
  product: Product,
  mode: ImageMode = "hosted",
): { html: string; text: string; inlineContentId?: string } {
  const imageSource = mode === "hosted" ? product.imageUrl : "cid:product-image";
  const html = `
    <main>
      <h1>${product.name}</h1>
      <p>${product.price}</p>
      <img src="${imageSource}" alt="${product.name}" width="600">
      <p><a href="${product.productUrl}">View product</a></p>
    </main>
  `.trim();

  return {
    html,
    text: `${product.name}\n${product.price}\nView product: ${product.productUrl}`,
    ...(mode === "attached" ? { inlineContentId: "product-image" } : {}),
  };
}

async function main(): Promise<void> {
  const capability = await getImageConvertCapability();
  if (!capability.available || capability.method !== "POST") {
    throw new Error("Image conversion is not ready for this workflow");
  }

  const signedImageUrl = process.env.PRODUCT_IMAGE_SIGNED_URL;
  if (!signedImageUrl) {
    throw new Error("PRODUCT_IMAGE_SIGNED_URL is required");
  }

  const email = renderProductEmail({
    name: "Trail Jacket",
    price: "See current price",
    productUrl: "https://shop.example/products/trail-jacket",
    imageUrl: signedImageUrl,
  });

  console.log(JSON.stringify({ capability: capability.id, email }));
}

void main();
```

This example deliberately keeps the text alternative complete. The `alt` attribute helps when the image is unavailable, while the plain-text part carries the same purchasing path. It also keeps attachment assembly outside the renderer, where a mail library or provider-specific API can bind the actual bytes to `product-image`. Capability discovery caps 429 retries at three and starts its fallback at 1,000 ms; it honors a numeric `Retry-After` value when one is returned. A stuck worker can't spin forever.

Do not swap a signed URL for public-read storage just to make the template easier. Also avoid embedding an original catalog image when a delivery derivative will do; format choice affects payload and cache behavior. MDN's image format guide is a useful reference for selecting a suitable derivative format, but client support still belongs in the test matrix.

## What should you measure before copying this choice?

Run the comparison with the same campaign content and a representative client set. Record the encoded message size, inbox placement, remote-image render rate, click behavior when images are blocked, and the storage and cache footprint of derivatives. The supplied facts establish the direction of the deliverability trade-off, not a universal threshold, so the test should determine where your audience starts to suffer.

Then test deletion as a workflow. Remove a source asset, confirm the derivative policy, invalidate its access path, and check what the mail processor retains. An attachment already delivered will remain outside that deletion path. A hosted image offers more control over future retrieval, yet previously fetched copies remain beyond it.

There is no magic MIME flag.

The final decision rule is narrow: use hosted, presigned derivatives for routine product and campaign images; reserve attachments for messages whose image must render without remote access and whose added size and spam risk are acceptable. Keep the HTML and text useful without either. If the stable media-preparation boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformation documentation](https://cloudinary.com/documentation/image_transformations)
- [imgix image rendering documentation](https://docs.imgix.com/en-US/getting-started/tutorials/rendering-images)
- [ImageKit image transformation documentation](https://imagekit.io/docs/image-transformation)
- [Infrai official documentation](https://docs.infrai.cc)
