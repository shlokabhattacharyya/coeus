# coeus 

i wanted to build an interactive notetaking tool with a mind mapping functionality similar to milanote and a constellation visual similar to obsidian but with an added layer of semantic mapping because i didn't like that the other two don't have that. also i like building my own versions of products that are already good.

## the math behind semantic mapping

### 0. notation

basic recurring notation throughout the following sections.

| symbol | meaning |
| --- | --- |
| $R = \{r_1,...,r_N\}$ | the set of readings being mapped |
| $x_i$ |  |
| $E: \Sigma^{*} \rightarrow \mathbb{R}^d$ | the embedding model, where $768 \leq d \leq 3{,}072$ |
| $\mathbb{S}^{d - 1} = \{ x \in \mathbb{R}^d : \lVert x \rVert = 1 \}$ | the unit sphere |
| $u_i \in \mathbb{S}^{d - 1}$ | the unit-length embedding of $r_i$ |
| $N_k (i)$ | the $k$-nearest neighbors of $i$ in embedding space |
| $y_i \in \mathbb{R}^2$ | the "world" position of $r_i$ on the map |
| $\mathcal{C}$ | the set of sections |

### 1. embedding

**definition 1.1 (input text):** each reading's input text is fixed as

$$
x_i = \texttt{title}_i + \texttt{"\\ n"} + \texttt{summary}_i + \texttt{"\\ n"} + \texttt{tags}_i.
$$

notes are excluded by design. as a result, editing a note never changes the position of a dot.

**definition 1.2 (normalized embedding):**

$$
v_i = E(x_i), \qquad u_i = \frac{v_i}{\lVert v_i \rVert}.
$$

**invariant 1.3**: vectors from different models live in unrelated spaces and cannot be compared. therefore, a map is valid only if every $u_i$ in it came from the same model. 

### 2. similarity and distance

**definition 2.1 (similarity):** for any vectors $a, b$ with angle $\theta$ between them, $a^\top b = \lVert a \rVert \lVert b \rVert.$

because embeddings have unit length $\text{\S}~1$,

$$
s_{ij} = u_i^\top u_j = \text{cos} \theta_{ij} \in [-1, 1].
$$

similarity depends only on direction, which encodes meaning, and not on length.

**proposition 2.2**: for unit vectors, 

$$
\lVert u_i - u_j \rVert^2 = 2 - 2s_{ij}.
$$

proof. $\lVert u_i - u_j \rVert2 = \lVert u_i \rVert^2 + \lVert u_j \rVert^2 - 2 u_i^\top u_j = 2 - 2s_{ij}.$

**definition 2.3 (distance):** $\delta_{ij} = \lVert u_i - u_j\rVert = \sqrt{2 - 2s_{ij}}$.

**corollary 2.4:** let $\delta_{ij} = \lVert u_i - u_j \rVert$ with $\lVert u_i \rVert = \lVert u_j \rVert = 1$, and let $\theta_{ij} \in [0, \pi]$ be the angle between $u_i$ and $u_j$. then:

<ol type="a">
<li> $\delta_{ij} = \sqrt{2 - 2 \text{cos} \theta} = 2 \text{sin} \frac{\theta}{2}$. </li>
<li> for every $i$ and every pair $j, l$, $ s_{ij} > s_{i\ell} \iff \delta_{ij} < \delta_{i\ell}$. in particular, for any $k$, the $k$ readings most similar to $i$ are exactly the $k$ readings nearest to $i$. </li>
<li> $\delta$ is a metric on $\mathbb{S}^{d - 1}$; $1 - s$ is not. </li>

</ol>

proof.
<ol type="a">
<li> the first equality holds by proposition 2.2. for the second equality, $2 - \text{cos} \theta = 4 \text{sin}^2 \frac{\theta}{2}$, and $\text{sin} \frac{\theta}{2} \geq$ on $[0, \pi]$. </li>
<li> $g(s) = \sqrt{2 - 2s}$ is strictly decreasing on $[-1 ,1]$, since $g'(s) = \frac{-1}{\sqrt{2 - 2s}} < 0$ for $s < 1$, and $g$ is continuous at $s = 1$. </li>
<li> 
<p> $\delta$ is the Euclidean norm restricted to a subset of $\mathbb{R}^d$, so it inherits all the metric axioms. </p>
<p> as a note, we define what it means for $\delta$ to be a metric, and why $1 - s$ is not. </p> 
<p> a function $d$ is a metric if, for all points x, y, z: </p>
<ol type="i">
<li> $d(x, y) \geq 0$ and $d(x, y) = 0$ only when $x = y$.
<li> $d(x, y) = d(y, x)$. </li>
<li> $d(x, z) \leq d(x, y) + d(y, z)$ [the triangle inequality] </li>
</li>
<p> <b>$\delta$ is a metric:</b> $\delta$ is an ordinary straight-line distance, $\lVert x - y \rVert$, and a straight-line distanc satsifes all three axiom for every point in $\mathbb{R}^d$. the points $u_i$ are the some of the points in $\mathbb{R}^d$, namely the ones on the unit sphere. so a statement that holds for all points $\mathbb{R}^d$ automatically holds for any subset of them. therefore, $\delta$ restricted to the sphere is still a metric. </p>
<p> <b>$1 - \delta$ is not a metric:</b> we show a counterexample. take three points on the unit circle, $u_a = \left(1,0\right), u_b = \left(\frac{1}{2}, \frac{\sqrt{3}}{2}\right), u_c = \left(-\frac{1}{2}, \frac{\sqrt{3}}{2}\right)$. these points live in $\mathbb{R}^2$, but padding them with zeros places them in $\mathbb{R}^d$ for any $d \geq 2$ without changing any dot product. the three similarities as dot products are $s_{ab} = 1 \cdot \frac{1}{2} + 0 \cdot \frac{\sqrt{3}}{2} = \frac{1}{2}, s_{bc} = \frac{1}{2} \cdot \left(-\frac{1}{2}\right) + \frac{\sqrt{3}}{2} \cdot \frac{\sqrt{3}}{2} = \frac{1}{2}, s_{ac} = 1 \cdot \left(-\frac{1}{2}\right) + 0 = -\frac{1}{2}. then, the would-be distances are $1 - s_{ab} = \frac{1}{2}, 1 - s_{bc} = \frac{1}{2}, 1 - s_{ac} = \frac{3}{2}.$ however, the triangle inequality would require $\frac{3}{2} \leq \frac{1}{2} + \frac{1}{2} = 1,$ which is false. therefore, $1 - s$ fails axiom 3, and fails being a metric.</p> 
</ol>


### 3. the neighbor graph

**definition 3.1 (embedding matrix):** $U \in \mathbb{R}^{N \times d}$ is the matrix whose $i$-th row is ${u_i}^\top$.

**definition 3.2 (similarity matrix):**

$$
K = UU^\top \in \mathbb{R}^{N \times N}, \qquad K_{ij} = {u_i}^\top u_j = s_{ij}.
$$

$K$ is symmetric ($K_{ij} = K_{ji}$), and its diagonal is all $1$s, since each $u_i$ has unit length.

**definition 3.2 (neighbor order):** each reading has a unique permanent identifier $\text{id}(i) \in \mathbb{N}$. for a fixed reading $i$, order all other readings by 

$$
j \prec_i \ell \iff \delta_{ij} < \delta_{i\ell} \ \text{ or } \ \left(\delta_{ij} = \delta_{i\ell} \text{ and } \operatorname{id}(j) < \operatorname{id}(\ell)\right).
$$

$N_k(i)$ is the set of the first $k$ readings under $\prec_i$, and $\text{pos}_i(j) \in \{0,...k-1\}$ is $j$'s position in that list.

**remark 3.3:** since ids are unique, no two readings are tied under $\prec_i$. $N_k(i)$ is therefore uniquely determined by the embeddings and ids alone, independent of storage order, sorting algorithm, or parallelization. by corollary 2.4(b), ordering by decreasing $s_{ij}$ gives the same $N_k(i)$.
