This README is still under development. Points to be covered and TODOs. Hopefully this will get you pointed in the right direction to make the cases work.

- **Case STL Files**
  - Currently, only the Noctua A6x25 (60mm) fan lid is available in the STL directory. The second case (A4x10/40mm) still needs adjustments and testing. _Coming soon._
- **Materials to Use and Considerations**
  - Nylon composite materials have proven to work very well so far.
  - Weaker materials like PLA or PETG need threaded inserts to mount the lid properly. However, friction fit might be good enough for PLA if not frequently opened.
- **Fan Compatibility**
  - Both cases are designed with Noctua PWM 5V fans in mind that can connect directly to the PWM port on the board.
    - [Noctua Fan Pins/Colors](https://faqs.noctua.at/en/support/solutions/articles/101000081757-what-pin-configuration-do-noctua-fans-use-/) - This document should help figure out how the pins connect.
    - If you don't want to cut the stock fan, you would need some replacement connectors. A quick search for `Micro MX JST 1.25mm 3 Pin Male and Female Connector Kit with 28AWG 150mm Cable for PCB Wiring` should get you what you need.
    - Some very basic soldering would be required.
- **Assembly Notes**
  - The 60mm fan lid requires some slightly tricky cable routing and screw placement to get mounted securely. It shouldn't be too difficult to figure out though.
- **Suggestions / TODOs**
  - Fan mesh covers would be a nice addition. These should be pretty easy to mount on the 40mm fan lid. For the 60mm fan lid, some minor adjustments might be needed. _Suggestion/TODO._
