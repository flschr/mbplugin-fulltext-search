# Search page

Fork of Manton Reece's original Micro.blog search plugin — thanks to Manton for sharing the foundation.

This version differs from the upstream release by:
- Localising the entire search page for German, including a shortcode intro that surfaces the total years and posts on the site.
- Replacing the original static results list with a live search interface that highlights keywords, shows contextual snippets, and lists tag/category metadata alongside keyboard-friendly controls.
- Sanitising the archive JSON feed before indexing, stripping `<script>`, `<style>`, `<noscript>`, and other markup noise, and augmenting it with taxonomy fields for richer filtering.
- Hardening the loading flow so the search box stays disabled until the archive is ready and any errors are clearly reported.

The fork is built for my private Micro.blog setup and can be tested at https://fischr.org/search.
