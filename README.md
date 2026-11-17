# coeus 

i wanted to build an interactive notetaking tool with a mind mapping functionality similar to milanote and a constellation visual similar to obsidian but with an added layer of semantic mapping because i didn't like that the other two don't have that. also i like building my own versions of products that are already good.

## the math behind semantic mapping

### 0. notation

basic recurring notation throughout the following sections.

| symbol | meaning |
| $R = {r_1,...,r_N}$ | the set of readings being mapped |
| $x_i$ |  |
| $E: \sum^{*} \rightarrow \mathbb{R}^d$ | the embedding model, where $768 \leq d \leq 3,072$ |
| $\mathbb{S}^{d - 1} = \{ x \in \matbb{R}^d : \lVert x \rVert = 1 \}$ | the unit sphere |
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

proof. $\lVert u_i - u_j \rVert2 = \lVert u_i \rVert^2 + \lVert u_j \rVert^2 - 2 u_i^\top u_j = 2 - 2s_{ij}. \qed$

**definition 2.3 (distance):** $\delta_{ij} = \lVert u_i - u_j\rVert = \sqrt{2 - 2s_{ij}}$.

**corollary 2.4:** let $\delta_{ij} = \lVert u_i - u_j \rVert$ with $\lVert u_i \rVert = \lVert u_j \rVert = 1$, and let $\theta_{ij} \in [0, \pi]$ be the angle between $u_i$ and $u_j$. then:

\begin{enumerate}[a]
\item $\delta_{ij} = \sqrt{2 - 2 \text{cos} \theta} = 2 \text{sin} \frac{\theta}{2}$.
\item for every $i$ and every pair $j, l$, $ s_{ij} > s_{i\ell} \iff \delta_{ij} < \delta_{i\ell}$. in particular, for any $k$, the $k$ readings most similar to $i$ are exactly the $k$ readings nearest to $i$.
\item $\delta$ is a metric on $\mathbb{S}^{d - 1}$; $1 - s$ is not.
\end{enumerate}

proof.
\begin{enumerate}[a]
\item the first equality holds by proposition 2.2. for the second equality, $2 - \text{cos} \theta = 4 \text{sin}^2 \frac{\theta}{2}$, and $\text{sin} \frac{\theta}{2} \geq$ on $[0, \pi]$.
\item $g(s) = \sqrt{2 - 2s}$ is strictly decreasing on $[-1 ,1]$, since $g'(s) = \frac{-1}{\sqrt{2 - 2s}} < 0$ for $s < 1$, and $g$ is continuous at $s = 1$.
\item $delta$ is the Euclidean norm restricted to a subset of $\mathbb{R}^d$, so it inherits all the metric axioms. 
\end{enumerate}

### 3. the neighbor graph

**defition 3.1:** $N_k (i)$ is the set of the $k$ indices $j \neq i$ with the smallest $\delta_{ij}$