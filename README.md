## About

A hands-on tutorial on elliptic curves

### Lesson 0 - defining elliptic curves

Continuous case uses geogebra


```python
import plotly
```

```python
import numpy as np
```

```python
def elliptic(p, a, b):
    x = y = np.arange(p)
    xx, yy = np.meshgrid(x,y)
    return (np.mod(yy**2, p) - np.mod(xx**3+a*xx+b, p) == 0)*1.0
```

## Finite elliptic curves

```python
import plotly.graph_objects as go

fig = go.Figure(data=go.Heatmap(z=elliptic(37, a=0, b=7), colorscale='gray'), layout=dict(width=700, height=700))
fig
```

Note symmetry about $p/2$


## Algebraic sum

Given
$$P=(x_p, y_p)$$
$$Q=(x_q, y_q)$$
What is $P + Q = -R$?

We want the (modulus) line passing through $P, Q$. The slope $m$ of that line is given by

$$m = (y_p-y_q)(x_p-x_q)^{-1} mod(p)  \quad P!=Q$$
$$ (3x_P^2+a) (2y_P)^{-1} mod(p) \quad P = Q, $$
$$a = 7 \quad \text{for bitcoin}$$

With $m$ we can obtain the $R$:
$$ x_R = (m^2 - x_P - x_Q) mod(p) $$
$$ y_r = (y_p + m(x_R-x_P)) mod(p) $$

```python
# taken from https://stackoverflow.com/questions/4798654/modular-multiplicative-inverse-function-in-python
# and http://anh.cs.luc.edu/331/notes/xgcd.pdf
def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = egcd(b % a, a)
        return (g, x - (b // a) * y, y)

def xgcd(a, b):
    prevx, x = 1, 0
    prevy, y = 0, 1
    while b:
        q = a//b
        x, prevx = prevx - q*x, x
        y, prevy = prevy - q*y, y
        a, b = b, a % b
    return a, prevx, prevy
    
def modinv_2(a, b):
#         a = abs(a)%b # seems to be a bug because the next line allows negative assertion to pass
        if a < 0:
            a += b
            
        for m in range(b):
            if (a*m)%b == 1:
                return m
        return None
assert (modinv_2(13, 17)*13)%17 == 1
assert (modinv_2(-13, 17)*(-13))%17 == 1


def modinv(a, m):
    """multiplicative mod inverse of a mod m"""
#         if( n < 0 ) {
#             n = n + this.k;
#         }
    if a < 0:
        a = a + m
#     g, x, y = egcd(a, m)
    g, x, y = xgcd(a, m)
    if g != 1:
        raise Exception(f'modular inverse does not exist for {a} mod {m}')
    else:
        return x % m

assert (modinv(13, 17)*13)%17 == 1
assert (modinv(-13, 17)*-13)%17 == 1
```

```python
def add_mod(P, Q, p, a, b=None):
    """add points P and Q through their intserection with curve (p,a,b) b is unused
    
    based on https://github.com/andreacorbellini/ecc/blob/865b8f9e79ed94a9d004a9b553f4b3add55878be/interactive/ec.js#L1059
    """
    if P is None:
        return Q
    if Q is None:
        return P
    
    x_p, y_p = P
    x_q, y_q = Q
    
    if x_p != x_q: # distinct points
        m = (y_p-y_q)*modinv(x_p - x_q);
    else:
        if (y_p == 0)&(y_q == 0 ):
        # This may only happen if P=Q is a root of the elliptic
        # curve, hence the line is vertical.
            print('found root of elliptic')
            return None
        elif y_p == y_q:
        # The points are the same, but the line is not vertical.
#             print('points are the same')
            m = (3*x_p*x_p+a)*modinv(2*y_p, p)
    
        else:
        # The points are not the same and the line is vertical.
            print('found vertical')
            return None
    m = m%p

    x_r = (m*m - x_p - x_q)%p
    y_r = (m*(x_p-x_r) - y_p)%p
    
    if x_r < 0:
        x_r += p
    if y_r < 0:
        y_r += p

    return x_r, y_r
    

add_mod((22, 6), (22, 6), 37, 7)
```

```python
P_ = (3,6)
for i in range(10):
    print(i, P_)
    P_ = add_mod(P_, P_, p=97, a=2)
```

```python
def multiply_mod(P, n, p, a, b=None):
    """multiply P n times for curve (p,a,b)"""
    P_ = P + ()
    if n == 0:
        return 0
    elif n == 1:
        return P_
    for _ in range(n):
        P_ = add_mod(P_, P_, p, a) + ()
    return P_

multiply_mod((3, 6), 7, p=97, a=2)
```

```python
e2 = dict(p=97, a=2, b=3)
```

```python
for _ in range(10):
    print(f"{_}: {multiply_mod((3,6), _, **e2)}")
```

How many points are in the subgroup generated by P?

1. count all points on the curve N (see [Schoof's Algorithm](https://en.wikipedia.org/wiki/Schoof%27s_algorithm))
1. find all divisors of N
1. for every divisor of N, see if $nP=0$

```python
def order(field):
    """calculate the order of the field including the point at infinity"""
    return int(field.sum()+1)

order(elliptic(37, a=-1, b=3))
```

```python
def divisors(n):
    for i in range(1, int(n / 2) + 1):
        if n % i == 0:
            yield i
    yield n
```

```python
def subgroup_order(P, p, a, b):
    N = order(elliptic(p, a, b))
    for _ in divisors(N):
#         print(_)
        if multiply_mod(P, _, p=p, a=a) == P:
            break
    if _ == N:
        return N # already contains the point at infinity
    else:
        return _+1

subgroup_order((2,3), p=37, a=-1, b=3)
```

```python
order(elliptic(p=29, a=-1, b=1)) # is prime!
```

```python
subgroup_order((9,24), p=29, a=-1, b=1) # should be 37 instead of 38
```

```python
fig = go.Figure(data=go.Heatmap(z=elliptic(p=29, a=-1, b=1), colorscale='gray'),
                layout=dict(width=700, height=700))
fig
```

# choosing generator point P

Trick is 
1. given N choose the subgroup order n to be a prime divisor of N
1. compute cofactor h = N/n
1. choose random point P -- this could be fun!
1. compute G=hP
1. if G != 0, then n(hP)=0 and G is a generator of the whole curve
1. if G = 0, go back to (3)

```python
def is_prime(n):
    prime_flag = 0
      
    if(n > 1):
        for i in range(2, int(np.sqrt(n)) + 1):
            if (n % i == 0):
                prime_flag = 1
                break
        if (prime_flag == 0):
            return True
    return False
```

```python
def prime_divisor(n):
    """find the largest prime divisor of n"""
    for _ in list(divisors(n))[::-1]:
        if is_prime(_):
            return _

def cofactor(order, n):
    return int(order/n)

prime_divisor(42)
```

```python
e1 = dict(p=29, a=-1, b=1)
```

```python
order(elliptic(**e1))
```

```python
prime_divisor(37)
```

```python
cofactor(37, 37)
```

```python

N_ = order(elliptic(**e1))
N_
```

```python
H__ = (12, 8)

G__ = multiply_mod(H__, 37, **e1)
G__
```

```python
for _ in range(1, 38):
    print(multiply_mod(G__, _, **e1))
```

## Secret sharing

Since $H=dG$ is uninvertable, we can define a public/private key pairs as ($d_{priv}, H_{pub}$)

Suppose alice has public $H_a$ and private key $d_a$ and similarly for Bob $H_b$ and $d_b$. Alice and Bob may compute a shared secret $S = d_aH_b = d_bH_a = d_ad_bG$. This shared secret can be used by either party to encrypt blobs of data without fear of eavesdropping.

```python
d_a = 5
H_a = multiply_mod(G__, d_a, **e1)

d_b = 9
H_b = multiply_mod(G__, d_b, **e1)

S_b = multiply_mod(H_a, d_b, **e1) # Bob generates secret key using Alice's pub key
S_a = multiply_mod(H_b, d_a, **e1) # Alice generates secret key using Bob's pub key
assert S_a == S_b # check that Alice and Bob get the same result
S_a
```

## ECDSA

1. choose $k \in [1, n]$ for subgroup order $n$
1. calculate $P=kG$
1. r = P_x mod n # get x coordinate of P (unless $r=0$)
1. $s = k^{-1}(z+rd_A) modn$

```python
d_a = 5
H_a = multiply_mod(G__, d_a, **e1)
```

```python
from random import randrange

G_ = (9,24) # generator
n_ = subgroup_order(G_, **e1) # should be 37 for {'p': 29, 'a': -1, 'b': 1}

def get_rk(G_, n_, **ec):
    r = 0
    while r == 0:
        k = randrange(n_) # random in [1,n]
        P_ = multiply_mod(G_, k, **ec)
        r, _ = P_
    return P_, k

P_, k = get_rk(G_, n_, **e1)
r, _ = P_
print(r)
```

```python
k_inv = modinv(k, n_) # only works for n_ prime
k_inv
```

```python
z = randrange(n_) # represents hash with same bitlength as n_

s = (k_inv*(z+r*d_a))%n_

sig_az = (r,s) # alice's signature on z message
sig_az
```

To verify signature:

1. $u_1 = s^{-1}z \text{modn}$
1. $u_2 = s^{-1}r \text{modn}$
1. $P = u_1 G + u_2 H_a$

```python
u_1 = (modinv(s,n_)*z)%n_
u_1
```

```python
u_2 = (modinv(s,n_)*r)%n_
u_2
```

```python
multiply_mod(G_, u_1, **e1), multiply_mod(H_a, u_2, **e1)
```

```python
P_result = add_mod(multiply_mod(G_, u_1, **e1),
        multiply_mod(H_a, u_2, **e1),
        **e1)
if r != P_result[0]%n_:
    raise AssertionError(f"signature does not match! {r} != {P_result[0]}")
```

```python

```
