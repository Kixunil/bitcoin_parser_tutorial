Bitcoin parser tutorial
=======================

This is a tutorial that aims to teach you how to parse Bitcoin blockchain.
There are several reasons for you to learn how to parse Blockchain:

* Learn Bitcoin in-depth
* Analyze the Blockchain, make statistics and experiments
* Make your own wallet
* Create more exotic transactions (e.g. smart contracts)

Tools and requirements
----------------------

The tutorial contains examples in Rust and C++. While Rust is considered
preferred language (it's both fast and safe), C++ is provided for people
unfamiliar with Rust.

You will need at least few blk files created by Bitcoin Core. The aim is to
parse existing, verified data, not to write a full node.

To begin the tutorial, read the Into chapter.

Discalimer
----------

Do not try to implement full verification! You will most probably screw it up
and become out-of-consensus! If you want to work on full node, just join Core.

Contributions
-------------

Contributions are welcome, provided you realease them under WTFPL license.

Special areas of interest:

* Typo fixes and bug fixes
* Examples in new languages
* New content - parsing data that this tutorial doesn't cover.

License
-------

WTFPL 2.0

All code in tis tutorial is free software. It comes without any warranty, to
the extent permitted by applicable law. You can redistribute it and/or modify it
under the terms of the Do What The Fuck You Want To Public License, Version 2,
as published by Sam Hocevar. See http://www.wtfpl.net/ for more details.
