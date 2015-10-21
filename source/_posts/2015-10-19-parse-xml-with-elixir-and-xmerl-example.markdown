---
layout: post
title: "Parse XML with Elixir and Xmerl Example"
date: 2015-10-19 18:26
comments: true
categories:
  - parse
  - elixir
  - erlang
  - xml
  - xmerl
  - example
---

Here I want to share a little snippet which explains how to use [Xmerl](http://www.erlang.org/doc/man/xmerl.html) library to parse XML in Elixir project with explanation.
<!-- more -->

``` xml simple.xml
<?xml version="1.0" encoding="UTF-8"?>
<breakfast_menu>
        <food>
                <name>Belgian Waffles</name>
                <price>$5.95</price>
                <description>Two of our famous Belgian Waffles with plenty of real maple syrup</description>
                <calories>650</calories>
        </food>
        <food>
                <name>Strawberry Belgian Waffles</name>
                <price>$7.95</price>
                <description>Light Belgian waffles covered with strawberries and whipped cream</description>
                <calories>900</calories>
        </food>
        <food>
                <name>Berry-Berry Belgian Waffles</name>
                <price>$8.95</price>
                <description>Light Belgian waffles covered with an assortment of fresh berries and whipped cream</description>
                <calories>900</calories>
        </food>
        <food>
                <name>French Toast</name>
                <price>$4.50</price>
                <description>Thick slices made from our homemade sourdough bread</description>
                <calories>600</calories>
        </food>
        <food>
                <name>Homestyle Breakfast</name>
                <price>$6.95</price>
                <description>Two eggs, bacon or sausage, toast, and our ever-popular hash browns</description>
                <calories>950</calories>
        </food>
</breakfast_menu>
```

``` elixir xml_parser.exs
defmodule XMLParser do
  require Record
  Record.defrecord :xmlElement, Record.extract(:xmlElement, from_lib: "xmerl/include/xmerl.hrl")
  Record.defrecord :xmlText,    Record.extract(:xmlText,    from_lib: "xmerl/include/xmerl.hrl")

  def parse(file) do
    File.read!(file)
      |> scan_text
      |> parse_xml
  end

  def scan_text(text) do
    :xmerl_scan.string(String.to_char_list(text))
  end

  def parse_xml({ xml, _ }) do
    # single element
    [element]  = :xmerl_xpath.string('/breakfast_menu/food[1]/description', xml)
    [text]     = xmlElement(element, :content)
    value      = xmlText(text, :value)
    IO.inspect to_string(value)
    # => "Two of our famous Belgian Waffles with plenty of real maple syrup"

    # multiple elements
    elements   = :xmerl_xpath.string('/breakfast_menu//food/name', xml)
    Enum.each(
      elements,
      fn(element) ->
        [text]     = xmlElement(element, :content)
        value      = xmlText(text, :value)
        IO.inspect to_string(value)
      end
    )
    # => "Belgian Waffles"
    # => "Strawberry Belgian Waffles"
    # => "Berry-Berry Belgian Waffles"
    # => "French Toast"
    # => "Homestyle Breakfast"
  end
end
```

[Record](http://elixir-lang.org/docs/v1.0/elixir/Record.html) is required to interface with Erlang records.<br />
Two kinds of records are used: [xmlElement](https://github.com/otphub/xmerl/blob/master/include/xmerl.hrl#L74) and [xmlText](https://github.com/otphub/xmerl/blob/master/include/xmerl.hrl#L90).<br />
[:xmerl_scan.string](http://www.erlang.org/doc/man/xmerl_scan.html#string-1) - Parse string containing an XML document.<br />
[:xmerl_xpath.string](http://www.erlang.org/doc/man/xmerl_xpath.html#string-2) - Extracts the nodes from the parsed XML tree according to XPath.

