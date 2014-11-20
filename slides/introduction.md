# Front-End Optimization

### Lon Ingram

<div style="text-align: center;">
[http://lawnsea.com/jssummit-front-end-optimization](http://lawnsea.com/jssummit-front-end-optimization)
</div>



<div class="center-image">
![Bazaarvoice](./assets/bv-logo.svg)
</div>

Notes: 
- i work at bazaarvoice
- we provide ratings and reviews for a large swath of the web
- our core market is ecommerce


Firebird is a mature, complex, mission-critical application, implemented as a
single-page Backbone app running in our customers' websites.

Firebird serves over 150MM unique visitors per month.

Notes:
- Firebird runs in the our customer's web page (not in an iframe)
- This entails a great deal of trust
- page load/render is critical, so we do our best to stay out of the way
- but, we still have to be fast, b/c we want to ensure the user has the benefit
  of reviews when making purchasing decision
