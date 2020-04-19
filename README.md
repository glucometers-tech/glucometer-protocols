<!--
SPDX-FileCopyrightText: 2016 The Glucometer Protocols Authors

SPDX-License-Identifier: CC-BY-SA-4.0
-->

#   Glucometer Protocols

This repository contains description of protocols for various glucometers.

The actual information is in the src/ directory.

##  Build Instructions

With Docker Compose:

```bash
# Site:
$ docker-compose run --rm foliant make site
# PDF:
$ docker-compose run --rm foliant make pdf
```

With pip and stuff (requires Python 3.6+, Pandoc, and TeXLive):

```bash
$ pip install -r requirements.txt
# Site:
$ foliant make site
# PDF:
$ foliant make pdf
```

## License

This work is licensed under a [Creative Commons Attribution 4.0 International
License](https://creativecommons.org/licenses/by/4.0/).

Use the information you'll find in this page for whatever purpose you see fit,
but there is no warranty about the accuracy of this information.

Most of the information has been reverse engineerd by individual contributors,
see the individual page for details.
