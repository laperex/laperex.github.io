---
title: "Using Basys3 with Vitis & Vivado [2024]"
layout: post
---

## Getting Started

#### Difficulty Level: **EASY**

This guide will walk you through the process of setting up and running UART and GPIO projects on the Basys3 FPGA board using Vivado and Vitis. By leveraging the MicroBlaze soft processor, along with AXI GPIO and AXI UART modules, you’ll learn how to create an embedded FPGA system capable of handling both input/output operations and serial communication.

Whether you’re new to FPGA development or simply looking to enhance your understanding of MicroBlaze and AXI peripherals, this guide is designed to make the process simple and approachable.

We’ll cover:
1. Configuring the MicroBlaze processor in Vivado
2. Adding AXI GPIO and UART peripherals
3. Writing software to control GPIO and UART in Vitis
4. Testing the design on your Basys3 board

By the end, you’ll have a working project that combines both GPIO and UART functionalities, paving the way for more advanced FPGA designs.

<!-- Jekyll also offers powerful support for code snippets: -->

<!-- {% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %} -->

<!-- Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk]. -->

<!-- [jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/ -->
