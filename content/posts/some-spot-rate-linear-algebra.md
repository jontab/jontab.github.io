+++
title  = "Some Spot Rate Linear Algebra"
date   = "2025-05-01"
author = "Jon"
tags   = ["finance"]
katex  = true
+++

## Background

I've been taking this online course in corporate finance lately, and I figured I'd share some of my knowledge I've learned in my recent fixed income securities module.

## Fixed Income Securities

You’ve probably heard of fixed income securities⸺things like bills, notes, and bonds. When they’re issued by the U.S. government, they’re considered about as safe as it gets, because, well, the U.S. has never defaulted on its debt.

Think of a fixed income security as a formal IOU. The government borrows money from you, promises to pay it back later, and in the meantime, sends you regular interest payments (a.k.a. coupons). These are locked into the contract⸺unlike stock dividends, which can only be estimated.

Each fixed income security has three main parts:

- A *maturity* date (when you get your money back),
- A *coupon rate* (the interest you’re paid, usually yearly),
- A *face value* (typically $1,000 per bond).

The key difference between bills, notes, and bonds comes down to how long they last⸺and whether they pay a coupon. Bills are short-term (under a year) and don’t pay interest. You buy them at a discount⸺for example, pay $995 now and get $1,000 in six months.

Bonds and notes, on the other hand, do pay coupons. For example:

- Alice buys a 20-year Treasury bond with a 5% coupon and a $1,000 face value.
- Every year, she receives $50 in interest.
- In year 20, she gets her last $50 payment *plus* the original $1,000 back.

The U.S.-backed fixed income securities are sold by the U.S. Treasury, usually via auction. Primary dealers⸺big banks and financial institutions⸺bid on them, and their bids help shape one of the most important concepts in finance: the *yield curve*.

## Spot Rates

One way to think about the yield curve is through *spot rates*. A spot rate is the return you'd get by buying a zero-coupon bond (one with no interest payments) that matures in a specific year. It's the going price for locking your money up for that long.

Say the 2-year spot rate is 5%. That means if someone asks to borrow $1,000 for two years, they'd better pay you back at least $1,102.50⸺because that's what you could earn by putting your $1,000 into a 2-year zero-coupon bond and letting it grow at 5% a year.

To figure out what a bond is worth, you discount each future cash flow using it's corresponding spot rate.

$$\sum_{t=1}^{T} \frac{CF_t}{(1+r_t)^t}$$

So, going back to Alice’s bond:

$$CF_1 = \cdots = CF_{19} = 50,$$ and $$CF_{20} = 50 + 1000.$$

## Spot Rates - Linear Algebra Edition

There's some fun linear algebra that you can do to calculate spot rates. Assuming you want to calcalate up to the n-year spot rate, and you have n linearly independent bonds, you can form an n-system of equations with a unique solution. For example, let's calculate the 1, 2, and 3-year spot rates given the following bonds.

- Bond A has a face value of $100, a maturity date of 1 year, has zero coupons, and currently trades at $98.
- Bond B has a face value of $100, a maturity date of 2 years, has a 5% annual coupon rate, and currently trades at $102.
- Bond C has a face value of $100, a maturity date of 3 years, has a 10% annual coupon rate, and currently trades at $112.

To price a 3-year bond, we can instantiate the general formula above,

$$PV=\frac{CF_1}{(1+r_1)}+\frac{CF_2}{(1+r_2)^2}+\frac{CF_3}{(1+r_3)^3}.$$

To make this solvable using matrices, we define:

$$x_1=\frac{1}{(1+r_1)},$$
$$x_2=\frac{1}{(1+r_2)^2},$$
$$x_3=\frac{1}{(1+r_3)^3}.$$

Now our equations look like this:

$$98=100\cdot x_1+0\cdot x_2+0\cdot x_3,$$
$$102=5\cdot x_1+105\cdot x_2+0\cdot x_3,$$
$$112=10\cdot x_1+10\cdot x_2+110\cdot x_3.$$

Stack those into matrix form:

$$\begin{bmatrix}98\\\\102\\\\112\end{bmatrix}=\begin{bmatrix}100&0&0\\\\5&105&0\\\\10&10&110\end{bmatrix}\cdot\begin{bmatrix}x_1\\\\x_2\\\\x_3\end{bmatrix}.$$

If we call that coefficient matrix **A**, then we can calculate its inverse easily in Excel or Python and use it like so,

$$A^{-1}\cdot\begin{bmatrix}98\\\\102\\\\112\end{bmatrix}=\vec{x}.$$

Using Excel, we get,

$$\vec{x}=\begin{bmatrix}0.9800\\\\0.9248\\\\0.8450\end{bmatrix}.$$

Then, we solve for each spot rate individually given our definitions earlier. We get,

$$\vec{r}=\begin{bmatrix}0.0204\\\\0.0399\\\\0.0577\end{bmatrix}.$$

So, our 1-year spot rate is 2.04%, our 2-year spot rate is 3.99%, and our 3-year spot rate is 5.77%. Isn't that cool?

## A Note on Linear Independence

The requirement that those n bonds be linearly independent is crucial for determining a unique solution. Each independent bond effectively reduces the degrees of freedom by one. If one bond is simply a multiple of another, the system becomes underdetermined, and the solution forms a line. For instance:

- Bond A has a face value of $100, a maturity date of 1 year, pays zero coupons, and currently trades at $98.
- Bond B has a face value of $200, a maturity date of 1 year, pays zero coupons, and currently trades at $196.

From the 3-equation setup discussed earlier, we know that the 1-year spot rate is 2.04%. However, since these two bonds offer no information about the 2-year spot rate, that rate remains unconstrained⸺a free variable⸺and the solution is therefore a line.  If two bond entries are dependent instead, then we would form a plane (and so on).

## A Note on Arbitrage

If our system of equations is underconstrained, it means some of our bond entries must be scalar multiples of each other. But what happens if the system is overconstrained⸺that is, no set of spot rates can satisfy all the equations? Let’s consider adding a fourth bond to the previous 3-bond example:

- Bond D has a face value of $100, a maturity of 3 years, has a 7.5% annual rate, and currently trades at $108.

Let’s compute Bond D’s correct price using the spot rates we already know:

$$PV=\frac{7.5}{1+0.0204}+\frac{7.5}{(1+0.0399)^2}+\frac{107.5}{(1+0.0577)^3}=105.1255.$$

Interesting⸺the correct price is lower than its market price. This means we can short Bond D at $108 and synthetically replicate its cash flows using a combination of Bonds A, B, and C. To find the appropriate amounts, we

$$\begin{bmatrix}-98&-102&-112&-108\\\\100&5&10&7.5\\\\0&105&10&7.5\\\\0&0&110&107.5\end{bmatrix}\cdot\vec{z}=\begin{bmatrix}100\\\\0\\\\0\\\\0\end{bmatrix}.$$

We get,

$$\vec{z}=\begin{bmatrix}-0.7530\\\\-0.7530\\\\33.9985\\\\-34.7892\end{bmatrix}.$$

In other words, by shorting 0.7530 units of Bonds A and B, shorting 34.7892 units of Bond D, and buying 33.9985 units of Bond C, we walk away with $100 upfront and no future liabilities. This is a classic example of arbitrage⸺profiting instantly from mispriced securities. Of course, we wouldn’t be the only ones spotting this. Traders would rush to short Bond D, driving its price down to the fair value of 105.1255. In effect, the market eliminates overconstrained systems of equations through arbitrage.

### Side Note - Yield to Maturity

Sometimes it's helpful to tell another investor what the average return would be if a particular bond were held until maturity. That’s the purpose of the yield to maturity (YTM) metric. It’s defined as the value y that satisfies:

$$PV=\sum_{t=1}^{T}\frac{CF_t}{(1+y)^t}.$$

In other words, you don’t need to know the spot rates to price the bond⸺you're just using a single interest rate, which makes things simpler. That said, this doesn't mean spot rates don’t matter; in fact, the yield to maturity is influenced by them. A concise way to think about it is that the YTM acts like a weighted average, where the weights are based on the cash flows. Let’s look at an example and compute the yield to maturity for a specific bond.

> A 3-year Treasury note with a face value of $100 and a 5% coupon rate. The 1-year spot rate is 2%, the 2-year is 3%, and the 3-year is 4%.

The price of the bond is,

$$PV=\frac{5}{1+0.02}+\frac{5}{(1+0.03)^2}+\frac{105}{(1+0.04)^3}=102.9596.$$

Then, we solve for the yield in,

$$PV=\frac{5}{1+y}+\frac{5}{(1+y)^2}+\frac{105}{(1+y)^3}.$$

Excel is telling me that y is equal to 3.93%, which is very close to the spot rate at maturity of 4%. This makes sense because the two coupon payments of $5 are very small compared to the maturity payment of $105.

## Related Terms

- Price elasticity;
- Macaulay duration;
- Modified duration;
- Convexity.
