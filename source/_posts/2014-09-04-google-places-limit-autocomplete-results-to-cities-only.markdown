---
layout: post
title: "Google Places: limit autocomplete results to cities only, hints"
date: 2014-09-04 21:04
comments: true
categories: 
  - google
  - google places
  - javascript
  - jquery
  - coffeescript
  - autocomplete
---

If you ever used the Google Places API, then you're probably familiar with `types` option, which allows you to restrict results from autocomplete in some way. I was having a requirement to show there only cities in UK. So, first of all I tried `(cities)` type in conjunction with `componentRestrictions`:
<!-- more -->
``` ruby application.html.haml
...
= javascript_include_tag "http://maps.google.com/maps/api/js?v=3.13&sensor=false&libraries=places"
...
```
``` coffeescript google_places_autocomplete.js.coffee
$ ->
  initialize = () ->
    options = {
      language: 'en-GB',
      types: ['(cities)'],
      componentRestrictions: { country: "uk" }
    }

    input = $('.address-search')
    autocomplete = new google.maps.places.Autocomplete(input[0], options)

  google.maps.event.addDomListener(window, 'load', initialize)
```
In result I got following places picker:

{% img /images/google_places_types_cities.png %}

Good enough, but my goal was to show and put cities only and for now after picking a city I was getting "City, Country". The solution is quite simple: specify the country needed in google API call with `&region=` parameter:
``` ruby application.html.haml
= javascript_include_tag "http://maps.google.com/maps/api/js?v=3.13&sensor=false&libraries=places&region=UK"
```

{% img /images/google_places_cities_only.png %}


#### Other Hints

During search for this solution I saw many other tips for google places autocomplete. I'm gonna post them below, hope they'll be useful.


  * To get rid of the pins on the left side of the list add following css:
``` css 
.pac-icon {
  width: 0;
  background-image: none;
}
```

Result:

{% img /images/google_places_cities_only_no_pins.png %}

  * To handle list items returned by google API
``` coffeescript
document.addEventListener('DOMNodeInserted', (event) ->
  target = $(event.target)
  if target.hasClass('pac-item')
    # replacing "Some City United Kingdom" to "Some City" for each list element
    # warning: on click it will still autofill input with "Some City, United Kingdom"
    target.html(target.html().replace(/ United Kingdom<\/span>$/, "</span>"))
)
```                        


  * To handle place selection you should add an event listener. It's helpful, if you want, for example, to perform AJAX call, since regular `onChange` will not work.

``` coffeescript 
google.maps.event.addListener(autocomplete, 'place_changed', ->
  # Here you can use:
  # $('.your-input').val() to get the value from your input
  # autocomplete.getPlace() to get detailed information about place chosen
  
  # Below is the hack to replace input content from "City, Country" to "City"
  # when using 'types': ('cities') without specific region.
  # And although I don't recommend to use it, since it flashes while
  # changing an input content; maybe someone will need it for other purpose
  place = autocomplete.getPlace()
  if place.address_components
    city = place.address_components[0] && place.address_components[0].short_name || ''

    input.blur()
    setTimeout(
      -> input.val(city)
      10
    )
)
```



