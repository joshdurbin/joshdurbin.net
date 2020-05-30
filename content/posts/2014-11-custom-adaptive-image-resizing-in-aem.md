+++
date = "2014-11-10"
title = "Custom Adaptive Image Resizing in AEM"
tags = [ "aem" ]
+++

AEM/CQ has long had many avenues to rendering various versions of content
stored in its JCR repo. Many implementations of AEM take advantage of the
workflows API and custom functionality to build vast libraries of custom
content from origin content loaded into AEM/CQ's DAM system -- a sub-system, if
even that, of AEM.

With responsive web design in full force, though, there is often a requirement
for additional sizing options for images and sometimes those are dynamic, depending
on what type of device the call originates from. Additionally, depending on the type of organization
running or maintaining AEM, they might not want to clutter their repository
with additional copies of large binary assets.

Future versions of AEM are rumored to have APIs aimed at making this more attainable, but
I had an opportunity to build one for an existing client and will run through what was built...

## The solution

The solution consists of fixed sizes and modes, a selector enforcement filter,
and a servlet meant to do the actual rendering.

- A dynamic image rendering servlet (back)
- A filter meant to evaluate, structure, and safe guard against erroneous input (back)
- A few enums meant to provide strict options around sizing and quality of images (back)

### Sizing and quality

First off, the sizing and image quality options are hardcoded in Java Enums. They could
be made configurable, but for this implementation they are not.

The quality enum consists of 3 values; low, med, high which map the resampling quality
factors in the underlying, re-rendering servlet `AbstractImageServlet`:

```java
public enum ImageQuality {

  LOW(.5),
  MEDIUM(.75),
  HIGH(1);

  private final double quality;

  public double getQualityValue() {
    return this.quality;
  }

  private ImageQuality(final double quality) {
    this.quality = quality;
  }
}
```

... and the sizing here is constructed using the terminology "ViewPort". A "ViewPort" maps
an image quality, size, and re-sampling options to a name:

- thumb at 40px
- mobile portrait at 320px
- mobile landscape at 480px
- small tablet at 600px
- tablet portrait at 768px
- tablet landscape at 1024px
- desktop at 1280px

...with the code:

```java
public enum ViewPortType {

  THUMB("40", 40, ImageQuality.HIGH, true),
  MOBILE_PORTRAIT("320", 320, ImageQuality.LOW, true),
  MOBILE_LANDSCAPE("480", 480, ImageQuality.MEDIUM, true),
  SMALL_TABLET("600", 600, ImageQuality.MEDIUM, true),
  TABLET_PORTRAIT("768", 768, ImageQuality.MEDIUM, true),
  TABLET_LANDSCAPE("1024", 1024, ImageQuality.MEDIUM, true),
  DESKTOP("full", 1280, ImageQuality.HIGH, false);

  private final String selector;
  private final Integer width;
  private final ImageQuality defaultQuality;
  private final Boolean resample;

  public String getSelector() {
    return selector;
  }

  public Integer getWidth() {
    return width;
  }

  public ImageQuality getDefaultQuality() {
    return defaultQuality;
  }

  public Boolean getResample() {
    return resample;
  }

  private ViewPortType(String selector, Integer width, ImageQuality defaultQuality, Boolean resample) {
    this.selector = selector;
    this.width = width;
    this.defaultQuality = defaultQuality;
    this.resample = resample;
  }

  private static final List<ViewPortType> TYPES = Lists.newArrayList(ViewPortType.values());
  private static final List<ViewPortType> TYPES_REVERSED = Lists.reverse(TYPES);
  private static final List<Integer> ORDERED_WIDTHS = Lists.newArrayList(Iterables.transform(TYPES, new Function<ViewPortType, Integer>() {

    @Override
    public Integer apply(ViewPortType viewPortType) {
      return viewPortType.getWidth();
    }
  }));

  public static List<ViewPortType> getTypes() {
    return ImmutableList.copyOf(TYPES);
  }

  public static List<ViewPortType> getTypesReversed() {
    return ImmutableList.copyOf(TYPES_REVERSED);
  }

  public static List<Integer> getWidths() {
    return ImmutableList.copyOf(ORDERED_WIDTHS);
  }

}
```

The functional methods in these examples: `Iterables`, list functional transformations, etc... are provided by [Guava](https://github.com/google/guava).

### Request flows for images

The request flow exists such that clients make calls to retrieve particular assets, for
example: `/path/to/asset.adapt.full.jpg`. The request passes through your CDN subsystems and local caches
before it arrives at the AEM Publisher node where it passes through a Servlet Filter. The Servlet
Filter ensures the selector is valid and, if not, will find the closest size up or down to a match. If the filter
has to query for another size it will respond with a HTTP 301 (Moved Permanently) with the proper `Location` header set.
If all of this passes, Sling will take over and route the request to the Servlet configured to handle
resources with the selector `adapt`, etc... etc...

```
              GET
              /path/to/...
              asset.adapt.full.jpg
+----------+                 +------------+
|  client  +-----------------> caching    |
+----------+                 | subsystems |
                             +-----+------+
                                   |
                         +---------+
                         |
 +--------------------------------------------+
 | AEM Publish Instance  |                    |
 |                       |                    |
 | +---------------------v------------------+ |
 | | AdaptiveImageSelectorEnforcementFilter | |
 | +---------------------+------------------+ |
 |                       |                    |
 | +---------------------v------------------+ |
 | | AdaptiveImageServlet                   | |
 | +----------------------------------------+ |
 |                                            |
 +--------------------------------------------+
```

### Utility methods

A utility class `AdaptiveImageUtil` exists to provide consist scaling, selector scrutinization and selection across
the servlet and filter. Again, note the pseudo functional style by Guava. **Note:** The use of `Optional` in these
examples is achieved via Guava and not Java >= 8.

```
public class AdaptiveImageUtil {

  private AdaptiveImageUtil() {
  }

  public static ViewPortType getTypeFromSelector(final String selector, final ViewPortType viewPortType) {

  return Iterables.tryFind(ViewPortType.getTypesReversed(), new Predicate<ViewPortType>() {

    @Override
    public boolean apply(ViewPortType viewPortType) {

    return StringUtils.equalsIgnoreCase(selector, viewPortType.getSelector());
    }
  }).or(viewPortType);
  }

  public static ViewPortType getNearestTypeFromSelector(final String selector) {

  final ViewPortType type;

  if (StringUtils.isNotBlank(selector) && StringUtils.isNumeric(selector)) {

    final Integer width = Integer.valueOf(selector);
    final Integer index = Collections.binarySearch(ViewPortType.getWidths(), width);

    if (index > -1) {

    type = ViewPortType.getTypes().get(index);

    } else {

    final ViewPortType previous = ViewPortType.getTypes().get(Math.max(0, -index - 2));
    final ViewPortType next = ViewPortType.getTypes().get(Math.min(ViewPortType.getTypes().size() - 1, -index - 1));
    type = width - previous.getWidth() < next.getWidth() - width ? previous : next;
    }

  } else {

    type = ViewPortType.DESKTOP;
  }

  return type;
  }

  public static Layer scaleImage(final Image image, final Integer newWidth, final Integer newHeight, final Style style) throws RepositoryException, IOException {

  return scaleLayer(applyStyleDataToImage(image, style), newWidth, newHeight);
  }

  public static Layer scaleAsset(final Asset asset, final AssetHandler assetHandler, final Integer newWidth, final Integer newHeight) throws RepositoryException, IOException {

  return scaleLayer(new Layer(assetHandler.getImage(asset.getOriginal())), newWidth, newHeight);
  }

  public static Layer scaleLayer(final Layer layer, final Integer newWidth, final Integer newHeight) throws RepositoryException, IOException {

  final Layer scaledLayer;

  final Integer currentWidth = layer.getWidth();
  final Integer currentHeight = layer.getHeight();

  if (newWidth < currentWidth) {

    final Double widthRatio = (double) newWidth / (double) currentWidth;
    final Double heightRatio = (double) newHeight / (double) currentHeight;

    Integer setHeight = newHeight;
    if (setHeight == 0) {
    setHeight = (int) (currentHeight * widthRatio);
    }

    final Integer potentialScaledHeight = (int) (currentHeight * widthRatio);
    final Dimension newSize;
    if (potentialScaledHeight >= setHeight) {
    newSize = new Dimension(newWidth, potentialScaledHeight);
    } else {
    newSize = new Dimension((int) (currentWidth * heightRatio), setHeight);
    }

    scaledLayer = renderScaledImageOnLayer(layer, newSize, newWidth, setHeight);
  } else {

    scaledLayer = layer;
  }

  return scaledLayer;
  }

  public static Layer applyStyleDataToImage(final Image image, final Style style) throws RepositoryException, IOException {

  final Layer layer = image.getLayer(false, false, false);

  image.loadStyleData(style);
  image.crop(layer);
  image.rotate(layer);

  return layer;
  }

  private static Layer renderScaledImageOnLayer(final Layer layer, final Dimension scaledSize, final Integer newWidth, final Integer newHeight) {

  layer.resize(scaledSize.width, scaledSize.height);

  final Integer shiftX;
  final Integer shiftY;

  if (scaledSize.width != newWidth) {
    shiftX = Math.abs(scaledSize.width - newWidth) / 2;
    shiftY = 0;
  } else {
    shiftX = 0;
    shiftY = Math.abs(scaledSize.height - newHeight) / 2;
  }

  final Rectangle newDimensions = new Rectangle();
  newDimensions.setBounds(shiftX, shiftY, newWidth, newHeight);

  layer.crop(newDimensions);

  return layer;
  }
}
```

### Sling Filter

The filter for the adaptive resizing servlet will scrutinize inbound requests looking for any
matching the selector criteria. Any matching requests will have their request info passed
to the aforemention utility method. This method will attempt to match to configure viewports
and manipulate the request as needed.

```
@SlingFilter(order = -1)
public class AdaptiveImageSelectorEnforcementFilter implements Filter {

  private final String SELECTOR_SEPERATOR = ".";

  @Override
  public void init(FilterConfig filterConfig) throws ServletException {

  }

  @Override
  public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {

    final SlingHttpServletRequest request = (SlingHttpServletRequest) servletRequest;
    final SlingHttpServletResponse response = (SlingHttpServletResponse) servletResponse;

    final List<String> selectors = Lists.newArrayList(request.getRequestPathInfo().getSelectors());

    Optional<ViewPortType> redirectType = Optional.absent();

    if (selectors.contains(AdaptiveImageServlet.SELECTOR) && selectors.size() == 2) {

      final String widthSelector = Iterables.getLast(selectors);
      final ViewPortType viewPortType = AdaptiveImageUtil.getNearestTypeFromSelector(widthSelector);

      if (!viewPortType.getSelector().equals(widthSelector)) {

        redirectType = Optional.of(viewPortType);
      }
    }

    if (redirectType.isPresent()) {

      final StringBuilder redirectURL = new StringBuilder();

      redirectURL.append(request.getRequestPathInfo().getResourcePath());
      redirectURL.append(SELECTOR_SEPERATOR);
      redirectURL.append(AdaptiveImageServlet.SELECTOR);
      redirectURL.append(SELECTOR_SEPERATOR);
      redirectURL.append(redirectType.get().getSelector());
      redirectURL.append(SELECTOR_SEPERATOR);
      redirectURL.append(request.getRequestPathInfo().getExtension());

      response.setStatus(HttpServletResponse.SC_MOVED_PERMANENTLY);
      response.setHeader(HttpHeaders.LOCATION, redirectURL.toString());

    } else {

      filterChain.doFilter(servletRequest, servletResponse);
    }
  }

  @Override
  public void destroy() {

  }
}
```

### Sling Servlet

If the aforementioned servlet allows the result to passthru, the follow servlet
will be invoked. Here we inject the `AssetStore` (a DAM service) and pull back
some data. We lean heavily on the `AbstractImageServlet` (of the DAM API)
and override a few methods.

```
@SlingServlet(label = "Adaptive Image Servlet",
    description = "Adaptive Image Component Servlet",
    extensions = AdaptiveImageServlet.EXTENSION,
    selectors = AdaptiveImageServlet.SELECTOR,
    resourceTypes = "sling/servlet/default")
public class AdaptiveImageServlet extends AbstractImageServlet {

  private static final Logger LOG = LoggerFactory.getLogger(AdaptiveImageServlet.class);
  private static final long serialVersionUID = 42L;

  public static final String SELECTOR = "adapt";
  public static final String EXTENSION = "jpg";

  @Reference
  private AssetStore assetStore;

  @Override
  protected Layer createLayer(final ImageContext imageContext) throws RepositoryException, IOException {

    final Stopwatch stopwatch = Stopwatch.createStarted();

    final SlingHttpServletRequest request = imageContext.request;
    final ViewPortType viewPortType = getViewPortTypeFromSelectors(request.getRequestPathInfo().getSelectors());

    final Layer layer;

    if (imageContext.resource.adaptTo(Asset.class) == null) {
      layer = createLayerForImage(imageContext, viewPortType);
    } else {
      layer = createLayerForAsset(imageContext, viewPortType);
    }

    LOG.debug("Took {}ms to create layer for request URI: {}", stopwatch.elapsed(TimeUnit.MILLISECONDS), request.getRequestURI());

    return layer;
  }

  @Override
  protected void writeLayer(final SlingHttpServletRequest request, final SlingHttpServletResponse response, final ImageContext context, final Layer layer) throws IOException, RepositoryException {

    final ViewPortType viewPortType = getViewPortTypeFromSelectors(request.getRequestPathInfo().getSelectors());

    if (!WCMMode.DISABLED.equals(WCMMode.fromRequest(request))) {
      response.setHeader(HttpHeaders.CACHE_CONTROL, "no-cache, no-store, must-revalidate");
      response.setHeader(HttpHeaders.EXPIRES, "0");
    }

    if (layer != null) {
      writeLayer(request, response, context, layer, viewPortType.getDefaultQuality().getQualityValue());
    } else {
      response.sendError(404);
    }
  }

  @Override
  protected String getImageType() {
    return StandardImageHandler.JPEG_MIMETYPE;
  }

  private static ViewPortType getViewPortTypeFromSelectors(final String[] selectors) {
    final List<String> listOfSelectors = Arrays.asList(selectors);
    final ViewPortType viewPortType = listOfSelectors.size() == 2 ? AdaptiveImageUtil.getTypeFromSelector(Iterables.getLast(listOfSelectors), ViewPortType.DESKTOP) : ViewPortType.DESKTOP;

    return viewPortType;
  }

  private Layer createLayerForAsset(final ImageContext imageContext, final ViewPortType viewPortType) throws NumberFormatException, RepositoryException, IOException {

    final Asset asset = imageContext.resource.adaptTo(Asset.class);
    final AssetHandler assetHandler = assetStore.getAssetHandler(asset.getMimeType());

    if (!"image".equals(asset.getMimeType().split("/")[0])) {
      LOG.error("The asset is not an image");

      return null;
    }

    final Layer layer;

    if (viewPortType.getResample()) {
      layer = AdaptiveImageUtil.scaleAsset(asset, assetHandler, viewPortType.getWidth(), 0);
    } else {
      layer = new Layer(assetHandler.getImage(asset.getOriginal()));
    }

    return layer;
  }

  private Layer createLayerForImage(final ImageContext imageContext, final ViewPortType viewPortType) throws RepositoryException, IOException {

    final Layer layer;

    final Image image = new Image(imageContext.resource);

    if (image.hasContent()) {

      if (viewPortType.getResample()) {
        layer = AdaptiveImageUtil.scaleImage(image, viewPortType.getWidth(), 0, imageContext.style);
      } else {
        layer = AdaptiveImageUtil.applyStyleDataToImage(image, imageContext.style);
      }

    } else {

      LOG.error("The image associated with this page does not have a valid file reference; drawing a placeholder.");
      layer = null;
    }

    return layer;
  }

}
```

Such an approach allows for directed responses and control over what sizes the app server is allowed to render. Some implementations of a similar
framework allow for end-users to enter any number, typically for width, that controls the dynamic rendering.
Such a implementation would leave the infrastructure incredible vunerable to denial of service attacks, overflows, etc...