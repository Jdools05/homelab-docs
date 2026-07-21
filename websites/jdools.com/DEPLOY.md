# Sitemap Deployment for K8s

## Files Created

1. `sitemap-template.xml` - Main sitemap listing all your services
2. `robots-template.txt` - Robots.txt pointing to the sitemap and disallowing /api/

## How to Deploy

### Option 1: Static ConfigMap (Recommended)

```bash
# Create ConfigMaps for both files
kubectl create configmap landing-page-sitemap \
  --from-file=sitemap.xml=./sitemap-template.xml \
  --dry-run=client -o yaml >> k8s/landing-configmaps.yaml

kubectl create configmap landing-page-robots \
  --from-file=robots.txt=./robots-template.txt \
  --dry-run=client -o yaml >> k8s/landing-configmaps.yaml
```

Then add to your `configmap.yaml` or keep separate.

### Option 2: Serve Directly from Existing ConfigMap

If you want to serve these from the existing `landing-page-html` ConfigMap, you'd need to modify the HTML to redirect or embed them, which is less clean. The ConfigMap approach above is cleaner.

## Ingress Configuration

Your Traefik ingress should already route `/` to the landing page service. For static files like sitemap.xml and robots.txt, you have two choices:

1. **Serve from same service** - Add the files to the existing ConfigMap or mount them as volumes
2. **Separate endpoint** - Create a new path in your ingress specifically for these files (less common)

For simplicity, serve them from the same landing-page service.

## Next Steps

1. **Deploy**: Apply the ConfigMaps and ensure they're accessible at:
   - `https://jdools.com/sitemap.xml`
   - `https://jdools.com/robots.txt`

2. **Verify**: Visit both URLs in a browser to confirm they're served correctly

3. **Submit to Search Engines** (important!):
   - Google Search Console: https://search.google.com/search-console
     - Add property: `https://jdools.com`
     - Verify ownership
     - Submit sitemap URL: `https://jdools.com/sitemap.xml`
   - Bing Webmaster Tools: https://www.bing.com/webmasters
     - Similar process

4. **Monitor**: Check indexing status in both consoles after a few days

## Updating the Sitemap

As you add new pages to any of your services, update `sitemap-template.xml`:
- Add `<url>` entries for each new page
- Update `lastmod` dates when you make changes
- Adjust `changefreq` and `priority` as appropriate
