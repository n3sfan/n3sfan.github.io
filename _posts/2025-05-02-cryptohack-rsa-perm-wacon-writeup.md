---
layout: post
title: Cryptohack RSA Perm (WACON) writeup
date: 2025-05-02 21:52 +0700
author: nesfan
description: LLL for adults.
---

TLDR: lấy lại $$d_p, d_q, k_p, k_q$$ bằng cách dùng LLL (Hidden Number Problem).

$$\newcommand{\norm}[1]{\left\lVert#1\right\rVert}$$

## Cách cũ
Trước khi đi vào cách, mình sẽ nói tại sao cách cũ (cũng HNP) mà mình không thành công. Sẽ có một số insight thú vị. Trong mật mã thì quan trọng nhất là tại sao lại hoạt động mà phải không?

Vì các chữ số hex của $d_p, d_q$ bị hoán vị nên có thể nói ta sẽ tìm hoán vị $$p_0, p_1, \ldots, p_{F} (0 \leq p_i \leq F)$$ và các $$p_i$$ khác nhau.

Gọi $$P_i (0 \leq i \leq F)$$ là các vị trí mà chữ số hex $$i$$ xuất hiện trong $$d_p$$. Gọi $$Q_i$$ tương tự với $$d_q$$.

Ta có 2 phương trình:

\begin{align}
    d_p &= p_0 \sum_{i \in P_0} 16^i + p_1 \sum_{i \in P_1} + \ldots p_{F} \sum_{i \in P_{F}}\\
    d_q &= p_0 \sum_{i \in Q_0} 16^i + p_1 \sum_{i \in Q_1} + \ldots p_{F} \sum_{i \in Q_{F}}
\end{align}

Biết $$e = 293$$:
$$\begin{cases}
    e d_p - 1 &= k_p (p - 1) \\
    e d_q - 1 &= k_q (q - 1)
\end{cases} $$ (I)

Thử đi hướng brute $$k_p, k_q$$, vì $$k_p, k_q \approx (e = 293)$$. Đây là hệ tuyến tính nên hãy thử LLL xem? Xây dựng lattice $$L$$ giống HNP.

$$L = \begin{bmatrix}
 k_p & & \\
     & k_q & \\
 \sum_{i \in P_0} & \sum_{i \in Q_0} & 1 \\ 
 \sum_{i \in P_1} & \sum_{i \in Q_1} & & 1\\
 \vdots & \vdots & & & \ddots \\
 \sum_{i \in P_F} & \sum_{i \in Q_F} & & & &1 \\
\end{bmatrix}$$

Kì vọng LLL sẽ ra vectơ $$sol = (1,1,p_0,p_1,\ldots,p_{15}) \in L$$. $$\norm{sol}$$ khoảng 5 bits.

Không được. Có thể là do $$\det{L}$$ là $$(k_p k_q)^{\frac{1}{18}} \approx (e^2)^{\frac{1}{18}}$$ khoảng 2 bits < 5.

Quan sát ma trận $$L$$ có các hàng có norm không đều vì $$P_i, Q_i$$ ở mỗi hàng tuỳ vào hoán vị $$p_i$$. Có cách scale, hoặc là nhân cho 1 số lớn cho hệ, mình đã thử và không được.

Thật ra thì mình nghĩ ban đầu không LLL hay BKZ ra dù scale, recenter $$p_i, \ldots$$ gì cũng trivial. Mình cũng không thấy trong paper của HNP có đề cập tới các cách này.

Và từ $$L$$, mình nghĩ ra ngay 2-3 cách tạo lattice khác trên ẩn $$p_i$$, nhưng những lattice này khá tệ. Để ý $$L$$ không dùng $$N$$. Mình đã thử brute 100 bits MSB của $$p, q$$ thì mới ra $$sol$$ (tất nhiên 2 phần tử đầu cộng thêm LSB $$p, q$$), cách này thì dẫn tới $$\norm{sol} > $$ 1 số basis trong $$L'$$.

Túm cái váy lại, tại sao lại không hoạt động? Trong paper HNP thấy Minkowski bound không thoả là khá chắc 90\%.


## Cách được
Mình không chắc cách này có phải intended? Vì flag dẫn mình đến 1 thứ gì đó...

$$
(\text{I})
\iff
\begin{cases}
    e d_p + k_p - 1 &= k_p p \\
    e d_q + k_q - 1 &= k_q q
\end{cases}
$$

$$
\begin{align*}
\Rightarrow
f &= (e d_p + k_p - 1)(e d_q + k_q - 1) = k_p k_q N\\
\Rightarrow
f &=    (e d_p + k_p - 1)(e d_q + k_q - 1) \equiv 0 \mod N\\
\end{align*}
$$

f có dạng:

$$
\begin{align*}
  f &= a_{F,F} p_{F}^2 + \ldots + a_{0,0} p_0^2 \\
    &+ \sum_i a_{kp,i} k_p p_i + \sum_i a_{kq,i} k_q p_i\\
    &+ a_{kp,kq} k_p k_q \\
    &+ \sum_i a_i p_i \\
    &+ a_{kp} k_p + a_{kq} k_q\\
    &+ a + 2^{18}
\end{align*}
$$

Cộng thêm $$2^{18}$$ vào f. Linearize f được 188 ẩn.

Xây dựng HNP:

$$L = \begin{bmatrix}
 N &  \\
 a_{F,F} & 2^{10} \\ 
 \vdots & & \ddots \\
 a_{0,0} & & & 2^{10} \\
 a_{kp,0} & & & & 2^{5} \\ 
 \vdots & & & & & \ddots \\
 a_{kq,F} & & & & & & & 2^{5} \\
 a_{kp,kq} & & & & & & & & 1 \\
 a_0    & & & & & & & & & 2^{14} \\
 \vdots & & & & & & & & & & \ddots \\ 
 a_F & & & & & & & & & & & 2^{14} \\
 a_{kp} & & & & & & & & & & & & 2^{9} \\
 a_{kq} & & & & & & & & & & & & & 2^{9} \\
 a + 2^{18} & & & & & & & & & & & & & & 2^{18} \\
\end{bmatrix}$$

$$\Rightarrow sol = (2^{18}, 2^{10} p_F^2, \ldots, 2^{18})$$. $$\norm{sol} \approx 2^{20} \approx \det{L} \approx 2^{20}$$. Nhưng vẫn không CVP ra.

### Cải tiến
$$p_0 + p_1 + \ldots + p_F = 120$$, nên khử 1 ẩn $$p_0$$ ra khỏi $$d_p, d_q$$.  f còn 169 ẩn, nhưng cũng chưa đủ. Thật ra thì chỉ cần khử 1 ẩn $$p_1$$ ra khỏi f (còn 151 ẩn)

$$
\begin{align*}
  f = &a_{F,F} p_{F}^2 + \ldots + a_{2,2} p_0^2 \\
    &+ \sum_{i=2}^{F} a_{kp,i} k_p p_i + \sum_{i=2}^F a_{kq,i} k_q p_i\\
    &+ a_{kp,kq} k_p k_q \\
    &+ \sum_{i=2}^{F} a_i p_i \\
    &+ a_{kp} k_p + a_{kq} k_q\\
    &+ a + 2^{18}
\end{align*}
$$

và CVP của rkm + Flatter... là ra được $$p_i$$ cần tìm trong 3 phút. Có thể brute $$p_1$$ với 16 chữ số hex, tức là LLL 16 lần. Như vậy 50p là có thể tìm được đáp án. Có thể chạy thêm song song để giảm thời gian.

## Tại sao lại hoạt động ?

Thú thật, mình cũng không biết =))). Thường nếu HNP không thành công là do error quá lớn, hoặc số ẩn quá nhiều. Scale không giúp ích trong bài này (dù các hàng có norm không đều). Phân tích xác suất để bắt được $$sol$$ có trong paper Subset Sum, HNP thì không thấy =)). 

Thật ra bài này dùng hàng mạnh (CVP rkm), chưa thử LLL (Kannan embed).

Trong trường hợp này trước khi cải tiến, $$\norm{sol}$$ lớn hơn Minkowski bound. Vậy thôi.
