# A Note on the Structure of Azimuth

The Urbit address space is structured into galaxies, stars, planets,
moons, and comets. There is ample documentation available for galaxies,
stars, and planets, but relatively little on moons and comets and their
distribution. This document summarizes the allocation of address space.

To determine the sponsor of any point, use *++sein:title*:

```
(sein:title our now \~marzod)
```

## Galaxy

Galaxies span the first 2⁸ addresses of Azimuth. There are 256 (0xff)
stars. Galaxy names are suffix-only.

|              | First Address | Last Address |
| ------------ | ------------- | ------------ |
|  Decimal     | 0             | 255          |
|  Hexadecimal | 0x0           | 0xff         |
|  \@p         | \~zod         | \~fes        |

As galaxies have no sponsors, they instead have an IP address
determined by `gal.urbit.org` at port 13337+galaxy number.

At the current time, galaxies play the role of network peer discovery,
but at some future time this will fall to the stars instead.

## Star

Stars span the remaining addresses to 2¹⁶. There are thus 65,536 -
256 = 65,280 stars. Star names have prefix and suffix. They share the
suffix with their sponsoring galaxy.

|              | First Address | Last Address |
| ------------ | ------------- | ------------ |
|  Decimal     | 256           | 65.535       |
|  Hexadecimal | 0x100         | 0xffff       |
|  \@p         | \~marzod      | \~fipfes     |

A star's sponsor can be calculated as modulo 2⁸. The first star of
\~zod is 0x100 \~marzod. The last star of \~zod is 0xffff -- 0xff =
\~fipzod. The last star (of \~fes) is 0xffff \~fipfes.

## Planet

Planets span the remaining addresses to 2³². There are thus
4,294,967,296 - 65,536 = 4,294,901,760 planets. Planet names occur in
pairs separated by a single hyphen. A planet's name is obfuscated so it
is not immediately apparent who its sponsor is.

|              | First Address | Last Address |
| ------------ | ------------- | ------------ |
|  Decimal     | 65.536        | 4.294.967.295 |
|  Hexadecimal | 0x1.0000      | 0xffff.ffff  |
|  \@p         | \~dapnep-ropmyl | \~dostec-risfen |

A planet's sponsor can be calculated as modulo 2¹⁶.

Galaxy planets occupy points beginning with 0x1.0000 \~dapnep-ronmyl
(for \~zod); \~zod's last galaxy planet is 0xffff.ffff -- 0xffff =
0xffff.0000 \~lodnyt-ranrud. The last galaxy planet (of \~fes) is
0xffff.ffff -- 0xffff + 0x100 = 0xffff.0100 \~hidwyt-mogbud.

Star planets span the remaining space. The first star planet (of
\~marzod) is 0x1.000 + 0x100 = 0x1.0100 \~wicdev-wisryt. The last star
planet (of \~fipfes) is 0xffff.ffff \~dostec-risfen. Remember that star
planet recur module 2¹⁶.

## Moon

Moons occupy the block to 2⁶⁴, with 2³² moons for each planet. Moon
names have more than two blocks (three or four) separated by single
hyphens.

|              | First Address | Last Address |
| ------------ | ------------- | ------------ |
|  Decimal     | 4.294.967.296 | 18.446.744.073.709.551.615 |
|  Hexadecimal | 0x1.0000.0000 | 0xffff.ffff.ffff.ffff |
|  \@p         | \~doznec-dozzod-dozzod | \~fipfes-fipfes-dostec-risfen |

Moons recur modulo 2³² from their sponsor. Thus dividing a moon's
address by 2³² and taking the remainder yields the address of the
sponsor.

Any moon that begins with the prefix \~dopzod-dozzod-doz\_\_\_ is a
galaxy moon, but not every galaxy moon begins with that prefix. The
first galaxy moon of \~zod is 0x1.0000.0000 \~doznec-dozzod-dozzod; the
last is 0xffff.ffff.ffff.ffff -- 0xffff.ffff =
\~fipfes-fipfes-dozzod-dozzod.

Any moon that begins with the prefix \~dopzod-dozzod-\_\_\_\_\_\_ is a
star moon (other than galaxy moons), but not every star moon begins with
that prefix. The first star moon of \~marzod is 0x1.0000.0000.0100
\~doznec-dozzod-dozzod-marzod; the last is 0xffff.ffff.ffff.ffff --
0xffff.ffff + 0x100 = 0xffff.ffff.0000.0000
\~fipfes-fipfes-dozzod-marzod.

Any moon from \~dopzod-\_\_\_\_\_\_-\_\_\_\_\_\_ onwards is a planet
moon.

## Comet

Comets occupy the upper portion of the Urbit address space. There are
approximately 3.4×10^38^ comets, a fantastically large number. Comet
names occur in blocks of five to eight, separated by a double hyphen.

|              | First Address | Last Address |
| ------------ | ------------- | ------------ |
| Decimal      | 18.446.744.073.709.551.616 | 340.282.366.920.938.463.463.374.607.431.768.211.456 |
| Hexadecimal  | 0x1.0000.0000.0000.0000 | 0xffff.ffff.ffff.ffff.ffff.ffff.ffff.ffff |
| \@p          | \~doznec\--dozzod-dozzod-dozzod-dozzod | \~fipfes-fipfes-fipfes-fipfes\--fipfes-fipfes-fipfes-fipfes |

A comet is sponsored by a star. Currently star sponsors are determined
randomly from a list supplied to `u3_dawn_come` in
`pkg/urbit/vere/dawn.c` from a jamfile provided by urbit.org at
`https://bootstrap.urbit.org/comet-stars.jam`.

Comets cannot be breached or rekeyed: possession of the comet is *ipso
facto* demonstration of ownership.
