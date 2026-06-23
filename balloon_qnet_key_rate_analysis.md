# Key Rate Calculation in `balloon_qnet`

In `balloon_qnet`, the key rate is not computed directly inside the free-space channel model. The code first simulates or estimates channel efficiency, then converts that efficiency into raw key rate and secret key rate.

## Core Formula

The main secret key rate (SKR) formula is:

```python
h(p) = -p * log2(p) - (1-p) * log2(1-p)

raw_rate = ratesources * sourceeff * efficiency
skr = raw_rate * (1 - 2 * h(QBER))
```

This formula is defined in `balloon_qnet/transmittance_simulation.py` and repeated in several `Studies/*.py` scripts.

Default parameters:

```python
ratesources = 80e6   # source repetition rate, Hz
sourceeff = 0.01     # source efficiency
QBER = 0.04
```

With `QBER = 0.04`:

```text
h(QBER) = 0.242292...
1 - 2h(QBER) = 0.515416...
SKR = 412332.5 * efficiency bit/s
```

For example, if `efficiency = 0.001`, then:

```text
SKR ~= 412.3 bit/s
```

## Inputs

The direct input to the SKR calculation is:

```python
efficiency
```

This is computed as:

```python
efficiency = received_count / sent_count
```

For example, in `balloon_qnet/transmittance_simulation.py`:

```python
sifted = Sifting(gs.QlientKeys[hap.name], hap.QlientKeys[gs.name])
efficiency = len(hap.QlientKeys[gs.name]) / max(1, len(gs.QlientKeys[hap.name]))
```

Although `sifted` is computed, the efficiency used for SKR is based on the number of received qubits divided by the number of sent qubits, not `len(sifted)`.

The upstream physical inputs that determine this efficiency include:

```text
direction: uplink or downlink
ground station altitude
HAP / balloon altitude
link distance
AO correction order
wavelength
initial beam waist W0
receiver aperture
obscuration ratio
Cn0 turbulence parameter
wind speed u_rms
pointing error
tracking efficiency
detector efficiency
atmospheric transmittance Tatm
simulation time
```

These parameters affect photon loss, which then affects the simulated receive ratio.

## Outputs

Different scripts return the rate in slightly different formats.

In `balloon_qnet/transmittance_simulation.py`:

```python
compute_skr(efficiency)
```

returns:

```text
secret key rate in bit/s
```

In `Studies/untrustedscenararchi1.py`, `Studies/untrustedscenararchi2.py`, `Studies/EntanglementStudy.py`, and `Studies/MDItest.py`, the typical return format is:

```python
return eff, rate, skr
```

where:

```text
eff  = channel efficiency
rate = raw key rate, bit/s
skr  = secret key rate, bit/s
```

In `Studies/trustedscenararchi2.py`, the function returns a dictionary of SKRs for multiple BB84 sublinks:

```python
{
    "AliceQonnSKR": skr1,
    "balloonQonn1SKR": skr2,
    "balloonQonn2SKR": skr3,
    "balloonballoonSKR": skr4,
    "BobQonnSKR": skr5
}
```

There the code divides by `1000`, so the output unit is:

```text
kbit/s
```

## How the Channel Model Feeds Into SKR

The lower-level free-space channel code computes transmittance or loss probability.

In `balloon_qnet/balloon_qnet/free_space_losses.py`:

```python
T = self._draw_channel_pdf_sample(length, n_samples)
prob_loss = 1 - T
```

Then `CachedChannel` applies those precomputed loss samples during the NetSquid simulation.

So the full chain is:

```text
physical link parameters
-> atmospheric/channel transmittance T
-> photon loss probability 1 - T
-> NetSquid qubit delivery success/failure
-> efficiency = received / sent
-> raw_rate = source rate * source efficiency * efficiency
-> SKR = raw_rate * (1 - 2h(QBER))
```

## Important Note

`QBER` is fixed as `0.04` in the SKR formula. The repository does contain QBER estimation functions, such as `estimQBER()` in `balloon_qnet/balloon_qnet/QEuropeFunctions.py`, but most SKR calculations do not use the simulated QBER dynamically. They use the constant value:

```python
QBER = 0.04
```
