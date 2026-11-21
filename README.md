# coeus 

i wanted to build an interactive notetaking tool with a mind mapping functionality similar to milanote and a constellation visual similar to obsidian but with an added layer of semantic mapping because i didn't like that the other two don't have that. also i like building my own versions of products that are already good.

## the math behind semantic mapping

### 0. notation

basic recurring notation throughout the following sections.

| symbol | meaning |
| --- | --- |
| $R = \{r_1,...,r_N\}$ | the set of readings being mapped |
| $x_i$ | the text that gets embedded for $r_i$ (title, summary, and tags) |
| $E: \Sigma^{*} \rightarrow \mathbb{R}^d$ | the embedding model, where $768 \leq d \leq 3{,}072$ |
| $\mathbb{S}^{d - 1} = \{ x \in \mathbb{R}^d : \lVert x \rVert = 1 \}$ | the unit sphere |
| $u_i \in \mathbb{S}^{d - 1}$ | the unit-length embedding of $r_i$ |
| $N_k (i)$ | the $k$-nearest neighbors of $i$ in embedding space |
| $y_i \in \mathbb{R}^2$ | the "world" position of $r_i$ on the map |
| $\mathcal{C}$ | the set of sections |
| $\mathcal{E}$ | the set of edges (lines on the map), each an unordered pair of readings $\lbrace i, j \rbrace$ |

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

**invariant 1.3:** vectors from different models live in unrelated spaces and cannot be compared. therefore, a map is valid only if every $u_i$ in it came from the same model.

### 2. similarity and distance

**definition 2.1 (similarity):** for any vectors $a, b$ with angle $\theta$ between them, $a^\top b = \lVert a \rVert \lVert b \rVert \cos \theta$.

because embeddings have unit length,

$$
s_{ij} = u_i^\top u_j = \cos \theta_{ij} \in [-1, 1].
$$

similarity depends only on direction, which encodes meaning, and not on length.

**proposition 2.2:** for unit vectors,

$$
\lVert u_i - u_j \rVert^2 = 2 - 2s_{ij}.
$$

*proof.* $\lVert u_i - u_j \rVert^2 = \lVert u_i \rVert^2 + \lVert u_j \rVert^2 - 2 u_i^\top u_j = 2 - 2s_{ij}$.

**definition 2.3 (distance):** $\delta_{ij} = \lVert u_i - u_j\rVert = \sqrt{2 - 2s_{ij}}$.

**corollary 2.4:** let $\delta_{ij} = \lVert u_i - u_j \rVert$ with $\lVert u_i \rVert = \lVert u_j \rVert = 1$, and let $\theta_{ij} \in [0, \pi]$ be the angle between $u_i$ and $u_j$. then:

(a) $\delta_{ij} = \sqrt{2 - 2 \cos \theta_{ij}} = 2 \sin \frac{\theta_{ij}}{2}$.

(b) for every $i$ and every pair $j, \ell$, $s_{ij} > s_{i\ell} \iff \delta_{ij} < \delta_{i\ell}$. in particular, for any $k$, the $k$ readings most similar to $i$ are exactly the $k$ readings nearest to $i$.

(c) $\delta$ is a metric on $\mathbb{S}^{d - 1}$; $1 - s$ is not.

*proof.*

(a) the first equality holds by proposition 2.2. for the second equality, $2 - 2\cos \theta = 4 \sin^2 \frac{\theta}{2}$, and $\sin \frac{\theta}{2} \geq 0$ on $[0, \pi]$.

(b) $g(s) = \sqrt{2 - 2s}$ is strictly decreasing on $[-1, 1]$, since $g'(s) = \frac{-1}{\sqrt{2 - 2s}} < 0$ for $s < 1$, and $g$ is continuous at $s = 1$. a strictly decreasing function reverses every strict inequality.

(c) a function $d$ is a metric if, for all points $x, y, z$:

- (i) $d(x, y) \geq 0$, and $d(x, y) = 0$ only when $x = y$;
- (ii) $d(x, y) = d(y, x)$;
- (iii) $d(x, z) \leq d(x, y) + d(y, z)$ (the triangle inequality).

*$\delta$ is a metric:* $\delta$ is an ordinary straight-line distance, $\lVert x - y \rVert$, and a straight-line distance satisfies all three axioms for every point in $\mathbb{R}^d$. the points $u_i$ are some of the points in $\mathbb{R}^d$, namely the ones on the unit sphere, so a statement that holds for all points in $\mathbb{R}^d$ automatically holds for any subset of them. therefore, $\delta$ restricted to the sphere is still a metric.

*$1 - s$ is not a metric:* we show a counterexample. take three points on the unit circle,

$$
u_a = \left(1, 0\right), \qquad u_b = \left(\frac{1}{2}, \frac{\sqrt{3}}{2}\right), \qquad u_c = \left(-\frac{1}{2}, \frac{\sqrt{3}}{2}\right).
$$

these points live in $\mathbb{R}^2$, but padding them with zeros places them in $\mathbb{R}^d$ for any $d \geq 2$ without changing any dot product. the three similarities are

$$
s_{ab} = 1 \cdot \frac{1}{2} + 0 \cdot \frac{\sqrt{3}}{2} = \frac{1}{2}, \qquad s_{bc} = \frac{1}{2} \cdot \left(-\frac{1}{2}\right) + \frac{\sqrt{3}}{2} \cdot \frac{\sqrt{3}}{2} = \frac{1}{2}, \qquad s_{ac} = 1 \cdot \left(-\frac{1}{2}\right) + 0 = -\frac{1}{2}.
$$

the would-be distances are then

$$
1 - s_{ab} = \frac{1}{2}, \qquad 1 - s_{bc} = \frac{1}{2}, \qquad 1 - s_{ac} = \frac{3}{2}.
$$

the triangle inequality would require $\frac{3}{2} \leq \frac{1}{2} + \frac{1}{2} = 1$, which is false. therefore, $1 - s$ fails axiom (iii), and is not a metric.

### 3. the neighbor graph

**definition 3.1 (embedding matrix):** $U \in \mathbb{R}^{N \times d}$ is the matrix whose $i$-th row is $u_i^\top$.

**definition 3.2 (similarity matrix):**

$$
K = UU^\top \in \mathbb{R}^{N \times N}, \qquad K_{ij} = u_i^\top u_j = s_{ij}.
$$

$K$ is symmetric ($K_{ij} = K_{ji}$), and its diagonal is all $1$s, since each $u_i$ has unit length.

**definition 3.3 (neighbor order):** each reading has a unique permanent identifier $\text{id}(i) \in \mathbb{N}$. for a fixed reading $i$, order all other readings by

$$
j \prec_i \ell \iff \delta_{ij} < \delta_{i\ell} \ \text{ or } \ \left(\delta_{ij} = \delta_{i\ell} \text{ and } \text{id}(j) < \text{id}(\ell)\right).
$$

$N_k(i)$ is the set of the first $k$ readings under $\prec_i$, and $\text{pos}_i(j) \in \lbrace 0, ..., k-1 \rbrace$ is $j$'s position in that list.

**remark 3.4:** since ids are unique, no two readings are tied under $\prec_i$. $N_k(i)$ is therefore uniquely determined by the embeddings and ids alone, independent of storage order, sorting algorithm, or parallelization. by corollary 2.4(b), ordering by decreasing $s_{ij}$ gives the same $N_k(i)$.

**definition 3.5 (displayed edges):** a line connects $i$ and $j$ if either one is among the other's neighbors:

$$
\lbrace i, j \rbrace \in \mathcal{E} \iff j \in N_k(i) \text{ or } i \in N_k(j).
$$

**remark 3.6:** edges are computed in $\mathbb{R}^d$, not on the 2D map.

### 4. projection to 2D (UMAP)

we want a matrix $Y \in \mathbb{R}^{N \times 2}$ with rows $y_i$ such that the readings that are neighbors on $\mathbb{S}^{d - 1}$ remain neighbors in the plane.

**definition 4.1 (nearest-neighbor distance):** for each reading $i$,

$$
\rho_i = \min_{j \in N_k(i)} \delta_{ij}.
$$

weights depend only on the excess distance $\delta_{ij} - \rho_i \geq 0$. this gives each reading a full-strength link to its nearest neighbor, so none are disconnected. it also corrects for distance concentration: in high dimensions, all of a reading's neighbors tend to lie at similar distances, so most of each distance is a shared amount that says nothing about which neighbor is closer. subtracting $\rho_i$ removes that shared amount, so the weights instead reflect how much further away each neighbor is than the closest one.

**definition 4.2 (local scale):** $\sigma_i > 0$ is the solution of

$$
F_i(\sigma) = \sum_{j \in N_k(i)} \exp \left(- \frac{\delta_{ij} - \rho_i}{\sigma}\right) = \log_2 k,
$$

where the left side is the total weight of $i$'s neighbors and the right side is the target. $\sigma_i$ sets how fast weight decays with excess distance. it is chosen per reading so that every reading's weights sum to the same total. the target $\log_2 k$ is UMAP's convention; any target strictly between $1$ and $k$ would also yield a valid $\sigma_i$.

**lemma 4.3:** let $m_i \geq 1$ be the number of neighbors at distance exactly $\rho_i$. if $m_i < \log_2 k$, then $\sigma_i$ exists and is unique.

*proof.* the $m_i$ tied terms equal $1$ for every $\sigma$. each other term increases strictly from $0$ to $1$ as $\sigma$ goes from $0$ to $\infty$. the sum therefore increases strictly from $m_i$ to $k$, and attains $\log_2 k$ exactly once.

$\sigma_i$ is computed by bisection. if $m_i \geq \log_2 k$ (duplicate summaries), $\sigma_i$ is set to a small floor.

**definition 4.4 (directed weights):** how strongly reading $i$ is connected to each other reading $j$, from $i$'s point of view, is

$$
w_{j | i} = \begin{cases}
    \exp\left(- \frac{\delta_{ij} - \rho_i}{\sigma_i}\right) & j \in N_k(i) \\
    0 & \text{otherwise.}
    \end{cases}
$$

**remark 4.5:** $w_{j | i}$ depends only on how $i$'s neighbors are spaced relative to its nearest one, not on absolute distance. an isolated reading therefore still has a clear set of near neighbors to stay close to on the map.

**definition 4.6 (symmetric weights):**

$$
w_{ij} = w_{j | i} + w_{i | j} - w_{j | i} w_{i | j} \in [0, 1].
$$

neighbor relations are not mutual, so an outlier may list a cluster member that does not list it back. reading each directed weight as the probability that one reading claims the other, $w_{ij}$ is the probability that at least one does. unlike an average, this keeps one-sided links at full strength, which keeps outliers attached to their nearest cluster. the matrix $W = (w_{ij})$ is symmetric.

**definition 4.7 (layout):** $Y$ is chosen so that readings with high weight $w_{ij}$ are pulled together and readings with $w_{ij} = 0$ are pushed apart. UMAP does this by measuring closeness on the map with a score $q_{ij} \in [0, 1]$ that decreases with $\lVert y_i - y_j \rVert$, and minimizing the mismatch between $q_{ij}$ and $w_{ij}$ over all pairs. the parameter `min_dist` sets how tightly points within a cluster may pack.

**remark 4.8 (optimization):** $Y$ is computed in three steps.

(a) **starting layout:* readings are first placed using a spectral embedding of $W$: the eigenvectors of a matrix built from $W$ that assign similar values to strongly linked readings. taking two of these eigenvectors as coordinates puts each cluster roughly in the right place before any refinement.

(b) *refinement:* the layout is then improved by stochastic gradient descent over 500 epochs. in each round, the following is repeated for every edge, more often for edges with higher $w_{ij}$:

- pull: move $y_i$ and $y_j$ slightly toward each other.
- push: pick 5 readings at random and move $y_i$ slightly away from each of them.

pushing away from a few random readings, rather than from every unlinked one, keeps each round fast; since most pairs are unlinked, a random handful is a good stand-in. the step size starts large and shrinks to zero, so early rounds set the overall shape and later rounds fine-tune it.

(c) *determinism:* the only randomness is which readings are picked in (b), and it is driven by a fixed random seed. together with remark 3.4, the same library always produces the same map.

**remark 4.9:** attraction acts only on pairs with $w_{ij} > 0$. consequently,

- local neighborhoods are preserved reliably;
- distances between clusters are only approximate;
- orientation carries no meaning, since the mismatch in definition 4.7 depends only on distances in $Y$ and is unchanged by rotating, reflecting, or translating it.