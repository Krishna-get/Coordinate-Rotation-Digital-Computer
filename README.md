# Theoretical Background for Asynchronous CORDIC Hardware Design

## Executive Summary

This document provides a rigorous and comprehensive theoretical foundation for the design and verification of an asynchronous CORDIC (COordinate Rotation DIgital Computer) hardware module using Rajit Manohar's ACT (Asynchronous Circuit Toolkit) toolchain. The work addresses fundamental concepts spanning three interdisciplinary domains: (1) the mathematical and computational principles of the CORDIC algorithm for transcendental function evaluation, (2) the design methodology, timing assumptions, and correctness guarantees of quasi-delay-insensitive (QDI) asynchronous circuits, and (3) the ACT toolchain's CHP (Communicating Hardware Processes) modeling framework for synthesizing asynchronous architectures. The document establishes the theoretical basis for why CORDIC is well-suited to asynchronous hardware implementation, how QDI design principles ensure correctness despite unbounded delays, and how the ACT toolchain facilitates the mapping of high-level behavioral specifications to synthesized asynchronous gate-level implementations.

---

## 1. Introduction and Motivation for Asynchronous CORDIC Design

### 1.1 Overview of Asynchronous Circuit Design

Asynchronous circuits represent a paradigm fundamentally distinct from the synchronous, clock-driven design methodology that has dominated digital electronics for decades. In synchronous circuits, a global clock signal orchestrates all state transitions, imposing artificial quantization on time and ensuring that all computation and communication occurs at fixed, clock-defined intervals. Conversely, **asynchronous circuits operate without a global clock**, allowing components to operate independently based on the completion of local computation and the exchange of handshaking signals.

The operational principle of asynchronous design is **event-driven**: state transitions occur only when data is available and downstream components are ready to accept results. This fundamental departure from clock-driven design yields several inherent advantages:

- **Elimination of clock distribution overhead**: No need for global clock routing, associated with increased power consumption, electromagnetic noise, and design complexity.
- **Automatic clock gating**: Components remain quiescent when not actively processing data, consuming power only proportional to actual workload rather than clock frequency.
- **Robustness to process, voltage, and temperature (PVT) variations**: Since circuit operation depends on actual delays rather than worst-case assumptions, designs naturally tolerate manufacturing variability and environmental fluctuations.
- **Modularity and composability**: Asynchronous circuits can be composed hierarchically without requiring global timing synchronization, enabling easier design reuse and system integration.
- **Fine-grain pipelining opportunities**: Data-driven execution allows multiple data tokens to flow through pipeline stages at variable rates, enabling higher throughput in applications with data-dependent computation.

However, asynchronous design introduces distinct challenges absent in synchronous methodology. Without a global clock to impose order, designers must explicitly reason about causality, temporal dependencies, and potential hazards arising from arbitrary, unbounded gate and wire delays. The fundamental challenge becomes: **How can we guarantee correct operation despite not knowing or controlling the actual delays in the circuit?**

### 1.2 The Challenge of Timing in Asynchronous Design

In synchronous design, the designer specifies a maximum clock period, and tools ensure that all combinational logic and interconnect delays fit within one clock cycle. Timing closure becomes a matter of either meeting or not meeting a fixed deadline. Asynchronous design inverts this problem: rather than imposing a fixed deadline, we must design circuits that operate correctly for *any* combination of gate and wire delays, within reasonable physical bounds.

This leads to several critical distinctions in asynchronous design methodology:

1. **Delay Models**: Synchronous design assumes bounded, predictable delays and requires them to be known at design time. Asynchronous design must accommodate:
   - **Fully delay-insensitive (DI)**: Circuits that operate correctly for unbounded delays on all gates and wires.
   - **Quasi-delay-insensitive (QDI)**: Circuits that operate correctly for unbounded delays except where explicitly relaxed via the **isochronic fork assumption**, a modest, physically realizable concession.
   - **Bundled data**: Circuits that use explicit delay lines to guarantee that control information arrives only after data has stabilized, blurring the boundary between synchronous and asynchronous methodologies.

2. **Synchronization via Handshaking**: In synchronous design, clock edges implicitly synchronize all circuit activity. In asynchronous design, handshaking protocols—based on explicit request and acknowledge signals—coordinate data movement between modules.

3. **Encoding Schemes**: To distinguish between valid data and the absence of data, and to enable delay-insensitive operation, special encoding schemes are employed:
   - **Dual-rail encoding**: Each logical variable is represented by two wires; specific patterns encode data or "spacer" (absence of data).
   - **Single-rail with explicit acknowledgment**: One data wire per variable, accompanied by separate control signals.

The design of the asynchronous CORDIC module will leverage **QDI design principles**, the strongest practical form of delay-insensitive design, combined with the ACT toolchain's support for formal specification, synthesis, and verification.

### 1.3 Why CORDIC is Suitable for Asynchronous Hardware

The CORDIC algorithm exhibits several properties that make it exceptionally well-suited to asynchronous implementation:

1. **Highly iterative structure**: CORDIC computation proceeds through a fixed sequence of iterations, each performing similar operations (rotation by micro-angle, coordinate update). This regularity maps naturally to asynchronous pipeline stages.

2. **Data-dependent iteration count**: In some variants (e.g., hybrid CORDIC or angle recoding CORDIC), the number of required iterations can vary based on input data. Asynchronous implementation naturally accommodates variable latency without the power penalty of synchronous clock gating.

3. **Shift-and-add arithmetic**: Each CORDIC iteration reduces to shifts (which can be implemented by wire routing), additions, and subtractions. These operations are efficiently realized in asynchronous form with minimal control overhead.

4. **Moderate throughput requirements**: CORDIC is typically used in signal processing and control applications with moderate data rates, where the variable latency of asynchronous design does not impose significant performance penalties.

5. **Energy efficiency**: For applications where input data arrives sporadically, asynchronous CORDIC naturally reduces power consumption by avoiding the switching activity of a global clock.

---

## 2. Fundamentals of the CORDIC Algorithm

### 2.1 Historical Context and Basic Concept

The CORDIC algorithm was invented by Jack Volder in 1959 for the UNIVAC ABACUS (later called the B58 program) with the explicit goal of computing trigonometric functions and coordinate transformations using only shifts, additions, and subtractions—avoiding expensive multiplication operations. The name CORDIC stands for **COordinate Rotation DIgital Computer**, capturing the essence of the algorithm: it performs coordinate rotations via a sequence of micro-rotations with carefully chosen angles.

The fundamental insight underlying CORDIC is elegant: instead of computing a general rotation by angle \(\theta\) via the standard rotation matrix involving sines and cosines (which would require multiplication or table lookup), we decompose the target rotation into a sum of micro-rotations by angles of the form \(\alpha_i = \arctan(2^{-i})\):

\[
\theta \approx \sum_{i=0}^{n-1} d_i \cdot \arctan(2^{-i})
\]

where \(d_i \in \{-1, +1\}\) are decision coefficients determined iteratively. Because each angle \(\arctan(2^{-i})\) has a tangent equal to a power of two, the rotation can be performed via shifts and additions, eliminating the need for multiplications of trigonometric values.

### 2.2 Circular CORDIC: Rotation Mode Mathematical Formulation

We begin with the standard 2D rotation transformation. Given an input vector \((x_0, y_0)\) and a target rotation angle \(\theta\), the rotated vector \((x_n, y_n)\) is given by:

\[
\begin{bmatrix} x_n \\ y_n \end{bmatrix} = \begin{bmatrix} \cos\theta & -\sin\theta \\ \sin\theta & \cos\theta \end{bmatrix} \begin{bmatrix} x_0 \\ y_0 \end{bmatrix}
\]

This can be rewritten in an angle-accumulator form. Let us define the rotation matrix for angle \(\alpha\) in a normalized form:

\[
R(\alpha) = \begin{bmatrix} \cos\alpha & -\sin\alpha \\ \sin\alpha & \cos\alpha \end{bmatrix}
\]

The key observation is that this can be factored as:

\[
R(\alpha) = \cos(\alpha) \begin{bmatrix} 1 & -\tan\alpha \\ \tan\alpha & 1 \end{bmatrix}
\]

The term \(\cos(\alpha)\) scales the result, while the matrix on the right performs the rotation. For the special angles \(\alpha_i = \arctan(2^{-i})\), the tangent is exactly \(2^{-i}\), so multiplication by \(\tan\alpha_i\) becomes a right shift by \(i\) bits.

In the iterative CORDIC algorithm for **rotation mode**, we accumulate rotation angle through a sequence of micro-rotations. The \(i\)-th iteration rotates the current coordinate pair \((x_i, y_i)\) by angle \(\alpha_i = \arctan(2^{-i})\) or its negation, depending on the sign of the remaining angle to rotate:

**Rotation Mode Iteration Equations:**

\[
x_{i+1} = x_i - d_i \cdot y_i \cdot 2^{-i}
\]

\[
y_{i+1} = y_i + d_i \cdot x_i \cdot 2^{-i}
\]

\[
z_{i+1} = z_i - d_i \cdot \arctan(2^{-i})
\]

where:
- \((x_i, y_i)\) are the current coordinates.
- \(z_i\) is the remaining angle to rotate (initially \(z_0 = \theta\)).
- \(d_i\) is the decision coefficient, chosen as:
  - \(d_i = -1\) if \(z_i < 0\) (we need to rotate clockwise).
  - \(d_i = +1\) if \(z_i \geq 0\) (we need to rotate counter-clockwise).

The scaling factor \(2^{-i}\) represents the tangent of the micro-angle, making the operation implementable via bit shifting.

After \(n\) iterations, the angle accumulator \(z_n\) approaches zero, and the coordinates \((x_n, y_n)\) approach the rotated vector. However, they are scaled by a constant factor:

\[
K_n = \prod_{i=0}^{n-1} \cos(\arctan(2^{-i}))
\]

This **CORDIC gain** is approximately \(1.6468\) for large \(n\), and it must be compensated for if accurate magnitude is required.

### 2.3 Circular CORDIC: Vectoring Mode

In **vectoring mode**, the algorithm performs the inverse operation: given a vector \((x_0, y_0)\), we rotate it until the \(y\)-component becomes zero (or very small), recording the angle of rotation required. This mode computes the **magnitude** and **phase angle** of the input vector.

**Vectoring Mode Iteration Equations:**

\[
x_{i+1} = x_i - d_i \cdot y_i \cdot 2^{-i}
\]

\[
y_{i+1} = y_i + d_i \cdot x_i \cdot 2^{-i}
\]

\[
z_{i+1} = z_i + d_i \cdot \arctan(2^{-i})
\]

where the decision coefficient is now chosen based on the sign of \(y_i\):
- \(d_i = -1\) if \(y_i < 0\) (rotate counter-clockwise to bring \(y\) toward zero).
- \(d_i = +1\) if \(y_i \geq 0\) (rotate clockwise to bring \(y\) toward zero).

After convergence, \(y_n \approx 0\), \(x_n \approx K_n \cdot \sqrt{x_0^2 + y_0^2}\) (the scaled magnitude), and \(z_n \approx \arctan(y_0/x_0)\) (the phase angle).

### 2.4 Transcendental Function Computation

The power of the CORDIC algorithm extends far beyond simple rotations. By initializing the vector and angle accumulator appropriately, CORDIC can compute a wide range of transcendental functions:

**Sine and Cosine (Rotation Mode):**
Initialize with \(x_0 = 1\), \(y_0 = 0\), and \(z_0 = \theta\). After rotation, \(\cos\theta \approx x_n / K_n\) and \(\sin\theta \approx y_n / K_n\).

**Arctangent (Vectoring Mode):**
Initialize with \(z_0 = 0\) and input vector \((x_0, y_0)\). After vectoring, \(z_n \approx \arctan(y_0 / x_0)\).

**Magnitude (Vectoring Mode):**
From the same vectoring operation, \(x_n \approx K_n \cdot \sqrt{x_0^2 + y_0^2}\).

**Hyperbolic Functions:**
By modifying the sequence of angles to use \(\alpha_i = \text{arctanh}(2^{-i})\) (where the angles repeat at certain indices), CORDIC can compute hyperbolic sine, cosine, and arctangent.

**Logarithm and Exponential:**
Through combinations of circular and hyperbolic CORDIC modes, logarithm and exponential functions can be computed.

### 2.5 Convergence Range and Theoretical Guarantees

For the **circular CORDIC** in rotation mode, the range of angles for which the algorithm converges is bounded. The convergence range is approximately:

\[
|\theta| \leq \sum_{i=0}^{\infty} \arctan(2^{-i}) \approx 1.7433 \text{ radians} \approx 99.9°
\]

For applications requiring full \(2\pi\) radians (\(360°\)), a preprocessing step rotates the input into the convergence range by a multiple of \(\pi/2\). This is typically implemented via a "negative iteration" that rotates by \(\pm\pi/2\) before the standard iterations.

**Theoretical convergence guarantees** rely on the monotonic decrease of the remaining angle magnitude at each iteration (for properly chosen decision coefficients) and the boundedness of the angle sequence. Formal analysis via recurrence relations shows:

\[
|z_{i+1}| < |z_i| - \epsilon_i
\]

for some positive \(\epsilon_i\), ensuring that \(z_n \to 0\) exponentially as \(n \to \infty\).

### 2.6 Fixed-Point Arithmetic Considerations

In hardware implementations, CORDIC operates on fixed-point representations rather than floating-point. The choice of fixed-point format (integer bits and fractional bits) directly impacts:

1. **Dynamic range**: The number of integer bits determines the magnitude of representable values.
2. **Accuracy**: The number of fractional bits determines precision; more fractional bits allow finer-grained representation and lower quantization error.
3. **Convergence**: Due to quantization, the iteration cannot achieve exact zero residual angle; instead, it converges to within the precision of the representation.

For a fixed-point number with \(Q\) fractional bits, the smallest representable magnitude is \(2^{-Q}\), and this becomes the effective convergence criterion. The error after \(n\) iterations is bounded by the magnitude of the \(n\)-th micro-angle plus quantization error.

In practice, for an \(m\)-bit input, \(n \approx m\) iterations are required to achieve \(m\)-bit precision in the output. Each iteration requires additions/subtractions and right shifts by \(i\) bits, making the computational complexity \(O(n)\) with \(n = O(m)\) for \(m\)-bit precision.

---

## 3. Asynchronous Circuit Design Fundamentals

### 3.1 Event-Driven Computation Model

The core of asynchronous design rests on an **event-driven computation model** fundamentally different from synchronous operation. In synchronous circuits, computation progresses according to clock edges occurring at regular time intervals. In asynchronous circuits, computation progresses according to **events**—signal transitions that indicate either data availability or completion of a computation stage.

The event-driven model has several key characteristics:

1. **No global timing reference**: There is no global clock; instead, local control logic determines when state transitions occur.
2. **Causality-based ordering**: The sequence of events is determined by actual causality (data dependencies) rather than a predetermined clock schedule.
3. **Handshake-based synchronization**: Communication between circuit components occurs via handshake protocols, where sender and receiver exchange request and acknowledge signals.
4. **Automatic latency adaptation**: Pipeline stages operate at rates determined by data availability and downstream readiness, not a fixed clock frequency.

A critical consequence of the event-driven model is that **the actual delay between successive operations is data-dependent and unpredictable at design time**. This is both an advantage (better adaptation to actual delays) and a challenge (requiring formal reasoning about causality).

### 3.2 Handshake Protocols: Four-Phase and Two-Phase

Handshaking is the mechanism by which asynchronous circuits coordinate activity. A handshake protocol involves two parties: a sender and a receiver, connected via channels. The sender indicates readiness to send data via a **request (req)** signal, and the receiver indicates readiness to accept via an **acknowledge (ack)** signal.

#### 3.2.1 Four-Phase Handshake (Return-to-Zero, RTZ)

The four-phase handshake is the most commonly used protocol due to its simplicity. It consists of four distinct phases per transaction:

1. **Phase 1 (Sender initiates)**: Sender asserts the request signal (req becomes 1). Data is presented on the data channel simultaneously or shortly after.
2. **Phase 2 (Receiver responds)**: Receiver detects the request, processes the data, and asserts the acknowledge signal (ack becomes 1).
3. **Phase 3 (Sender releases)**: Sender observes the acknowledge, knows the data has been received, and de-asserts the request signal (req becomes 0).
4. **Phase 4 (Receiver releases)**: Receiver observes the de-asserted request, finishes any required cleanup, and de-asserts the acknowledge signal (ack becomes 0). The system returns to the idle state, ready for the next transaction.

This protocol is "return-to-zero" because both signals return to zero (their idle state) after each transaction. The key property is **atomicity** of the handshake: either the protocol completes all four phases correctly, or a malfunction is detected.

The timing diagram for four-phase handshaking shows clear separation between phases: req↑ (req transitions low-to-high), ack↑ (ack transitions low-to-high), req↓, ack↓.

**Advantages of four-phase:**
- Robust to metastability due to clear separation of phases.
- Relatively straightforward to implement.
- Works with simple combinational logic.

**Disadvantages:**
- Four signal transitions per transaction may be slower than alternatives.
- Explicit return to zero requires additional time and energy.

#### 3.2.2 Two-Phase Handshake (Return-to-One, RTO)

The two-phase handshake achieves higher throughput by reducing the number of transitions per transaction:

1. **Phase 1 (Sender initiates)**: Sender transitions the request signal (toggling its current state). Data is presented.
2. **Phase 2 (Receiver responds)**: Receiver detects the request transition and responds by toggling the acknowledge signal.

After two signal transitions (as opposed to four), the protocol is ready for the next transaction. The next transaction involves toggling the request again, followed by the receiver toggling acknowledge.

**Advantages of two-phase:**
- Potentially higher throughput (two transitions instead of four).
- Lower energy per transaction in some cases.

**Disadvantages:**
- More complex control logic (must track state of the signals, not just their absolute values).
- Greater vulnerability to metastability issues.
- Requires careful design to avoid race conditions.

For the asynchronous CORDIC module, **four-phase handshaking** is typically preferred due to its robustness and straightforward implementation, though two-phase variants are possible for optimized designs.

### 3.3 Delay-Insensitive (DI) and Quasi-Delay-Insensitive (QDI) Design

#### 3.3.1 The Delay-Insensitive (DI) Ideal

A **fully delay-insensitive (DI)** circuit is one that operates correctly regardless of the delays on any gate or wire, provided delays are finite and positive. This is the strongest form of robustness and is the ideal target of asynchronous design. In a DI circuit, **no timing assumptions whatsoever** are imposed on the implementation.

However, fully DI circuits are severely restricted. Certain circuits (e.g., those with certain combinational logic patterns) cannot be implemented in a fully DI manner without unacceptable overhead or without relaxing some timing assumptions.

#### 3.3.2 The Isochronic Fork Assumption

To make practical asynchronous design feasible, **Alain Martin** introduced the concept of the **isochronic fork**, a minimal and physically realizable relaxation of the DI requirement. An isochronic fork occurs when a single signal transitions and drives multiple recipient gates or latches. The isochronic fork assumption states: **all recipients of a signal transition will observe the transition within some bounded (but small) time window**, as if the transition were simultaneous from their perspective.

Formally, if a wire forks into multiple branches at a junction, the isochronic fork assumption permits the designer to assert that the transition arrives at all branches within a sufficiently small time window that we can reason about it as simultaneous for the purpose of determining causality.

This assumption is **physically realizable** because:
1. In modern integrated circuits, typical wire lengths are short (micrometers), and signal propagation time is minimal.
2. The fork is typically confined to a small spatial area on the chip.
3. Physical placement and routing tools can control the relative delays of fork branches.

**Quasi-delay-insensitive (QDI)** circuits are those that operate correctly for unbounded gate and wire delays, except where isochronic fork assumptions are explicitly declared. QDI has proven to be the "Goldilocks" of asynchronous design: restrictive enough to enable efficient implementation, yet permissive enough to allow practical circuit design.

#### 3.3.3 Advantages of QDI Design

QDI design provides several concrete advantages:

1. **Robustness to process variations**: Since circuits do not rely on precise timing margins, manufacturing variations and environmental fluctuations do not compromise correctness.
2. **Modularity**: QDI modules can be composed without requiring global timing re-analysis.
3. **Energy efficiency**: Circuits avoid the overhead of synchronous clock distribution and operate at rates determined by actual data availability.
4. **Automatic latency adaptation**: The average latency of a pipeline naturally adapts to the actual computation time, without the need for explicit clock gating.

---

## 4. Encoding and Data Representation in QDI Circuits

### 4.1 Dual-Rail Encoding

In asynchronous circuits, we face a fundamental challenge: **How do we distinguish between valid data and the absence of data?** In synchronous circuits, the clock signal implicitly distinguishes these states (data is valid on clock edges, invalid otherwise). In asynchronous circuits, we need an explicit encoding mechanism.

The most widely used encoding in QDI design is **dual-rail encoding**. In this scheme, each logical variable is represented by **two physical wires** (rails):

For a logical variable \(V\) represented by rails \(V_1\) and \(V_0\):
- \((V_1, V_0) = (0, 1)\) encodes logical **0** (or false).
- \((V_1, V_0) = (1, 0)\) encodes logical **1** (or true).
- \((V_1, V_0) = (0, 0)\) represents the **spacer** state (absence of data).
- \((V_1, V_0) = (1, 1)\) is an **invalid state** and must never occur.

This creates a **one-hot encoding** where exactly one rail is high when data is valid, and both rails are low when the system is in the spacer (idle or reset) state.

### 4.2 Spacer States and Completion

The **spacer state** plays a crucial role in QDI operation. After data has been processed by a logic stage, the data must be returned to the spacer state before the next data token can be accepted. This ensures that:

1. **Unambiguous data boundaries**: Each data token is bracketed by spacer states, making it clear when one computation ends and the next begins.
2. **Cycle coherence**: In pipelined designs, the spacing of spacer states between data tokens ensures that multiple tokens do not collide in a single stage.
3. **Initialization and reset**: At startup, all signals are in the spacer state, providing a well-defined initial condition.

The sequence of states on a dual-rail encoded line in a four-phase handshake might be:

\[
\text{spacer} \to \text{data\_0} \to \text{spacer} \to \text{data\_1} \to \text{spacer} \to \ldots
\]

For example, for a single bit:
\[
(0,0) \to (1,0) \to (0,0) \to (0,1) \to (0,0) \to \ldots
\]

### 4.3 Completion Detection

**Completion detection** is the mechanism by which a circuit determines when a computation has finished and the output is valid. In synchronous design, the clock edge implicitly marks the end of computation. In asynchronous design, explicit completion detection is required.

A **completion detector** monitors the state of all dual-rail outputs from a combinational logic block and asserts a signal when all outputs have transitioned to valid data states (no rails in the spacer state). Formally, a completion detector asserts when:

\[
\text{complete} = \text{true} \iff \text{all output rails have transitioned to their next valid states}
\]

The completion detector output feeds into the **acknowledge (ack)** signal of the handshake protocol, signaling to the upstream stage that the data has been accepted and the stage is ready for new data.

For efficiency, completion detection often exploits problem-specific structure. In arithmetic operations, for instance, early completion can be detected before all bits have been computed, reducing latency.

---

## 5. The Muller C-Element and QDI Logic

### 5.1 Definition and Behavior

The **Muller C-element** (also called the C-gate, coincident element, or two-hand safety circuit) is the fundamental building block of asynchronous control logic and is essential to QDI design. Invented by David E. Muller in 1955, the C-element is defined by the functional equation:

\[
y_n = x_1 x_2 + (x_1 + x_2) y_{n-1}
\]

This corresponds to the following truth table for a two-input C-element:

| \(x_1\) | \(x_2\) | \(y_n\)       |
|---------|---------|---------------|
| 0       | 0       | 0             |
| 0       | 1       | \(y_{n-1}\)   |
| 1       | 0       | \(y_{n-1}\)   |
| 1       | 1       | 1             |

**Behavioral properties:**

1. **Activation (output goes high)**: The output transitions to 1 when **all inputs transition to 1** (or are already at 1). Specifically, if \(x_1 = 1\) and \(x_2 = 1\), then the output goes to 1.

2. **Deactivation (output goes low)**: The output transitions to 0 when **all inputs transition to 0** (or are already at 0). Specifically, if \(x_1 = 0\) and \(x_2 = 0\), then the output goes to 0.

3. **State-holding property**: If the inputs are different (\(x_1 \neq x_2\)), the output **retains its previous value** \(y_{n-1}\), neither transitioning high nor low. This is the crucial property that distinguishes the C-element from standard combinational gates.

The state-holding property gives the C-element its character as a **memory element**. In the intermediate state where inputs disagree, the output does not change, effectively memorizing the previous state.

### 5.2 Physical Implementation of the C-Element

A basic C-element can be implemented as cross-coupled NOR gates (for the "false" output) or cross-coupled NAND gates (for the "true" output), augmented with set and reset logic. However, naive implementations derived directly from the truth table do not reliably work due to **delay assumptions**.

For a C-element with inputs \(x_1\) and \(x_2\) and output \(y\), let:
- \(\delta_1\) = propagation delay from \(x_1\) to \(y\) via the internal feedback loop.
- \(\delta_2\) = propagation delay from \(x_2\) to \(y\) via the internal feedback loop.
- \(\Delta_1\) = propagation delay from \(x_1\) to \(y\) via the external environment.
- \(\Delta_2\) = propagation delay from \(x_2\) to \(y\) via the external environment.

For correct operation, the **internal feedback must be faster than the external path**:
\[\delta_1 < \Delta_1 \quad \text{and} \quad \delta_2 < \Delta_2\]

Practical implementations typically use:

1. **Cross-coupled latch-based C-element**: Uses an SR latch (set-reset latch) as the core state-holding element, with additional set and reset functions derived from the input signals. This approach is robust and widely used.

2. **CMOS transistor-level C-element**: Implemented using complementary pull-up and pull-down networks, providing fine-grained control over timing. More complex to design but potentially more efficient.

3. **Weak inverter feedback C-element**: Uses a weak inverter (high-resistance pull-up/pull-down) to maintain state while allowing direct transistor switching for faster transitions. Reduces area and power but requires careful biasing.

### 5.3 Role of the C-Element in QDI Correctness

The C-element is indispensable to QDI design for several reasons:

1. **Synchronization primitive**: The C-element acts as a **rendezvous** element, ensuring that multiple input signals have all transitioned before permitting output transitions. This is essential for synchronizing asynchronous processes.

2. **Isochronic fork management**: When a signal forks to multiple recipients, C-elements at each fork branch ensure that the transition is sensed by all branches before allowing dependent logic to proceed. This enforces the isochronic fork assumption locally.

3. **Hazard prevention**: The state-holding property prevents combinational hazards that would otherwise arise from intermediate transitional states. Without C-elements, the output might glitch when inputs transition in unexpected orders.

4. **Pipeline stage control**: In asynchronous pipelines, C-elements coordinate the four-phase handshake between adjacent stages, ensuring correct data flow and preventing data loss or corruption.

### 5.4 C-Element Extensions

For more complex scenarios, C-element variations are employed:

- **Weak C-element**: Relaxes the requirement that all inputs must transition; used in specific QDI templates.
- **C-element with set/clear (reset) signals**: Allows forced initialization to specific states, useful for circuit startup.
- **Multi-input C-element**: Generalizes to \(n > 2\) inputs, requiring all inputs to converge to the same value before the output transitions.

---

## 6. Asynchronous Datapath Design Principles

### 6.1 Dual-Rail Logic Circuits

Dual-rail logic is the primary approach for implementing combinational functions in QDI circuits. In dual-rail logic, each primary output is represented by two rails, and we must derive both the "true" output (when the logical value is 1) and the "false" output (when the logical value is 0).

For a Boolean function \(f(a, b, c, \ldots)\), we derive:
- **True function** \(f_1\): The normal Boolean expression for \(f\).
- **False function** \(f_0\): The Boolean expression for \(\overline{f}\) (the complement of \(f\)).

For example, for a two-input AND gate \(f = a \land b\):
- \(f_1 = a_1 \land b_1\) (both true rails must be high).
- \(f_0 = a_0 \lor b_0\) (at least one false rail must be high).

The dual-rail encoded output is \((f_1, f_0)\), which is valid when exactly one rail is high:
- Valid 1: \((f_1, f_0) = (1, 0)\)
- Valid 0: \((f_1, f_0) = (0, 1)\)
- Spacer: \((f_1, f_0) = (0, 0)\)

This approach ensures **delay-insensitive computation**: the output is valid only when both the true and false computations have completed, regardless of which path is faster.

### 6.2 Asynchronous Arithmetic Components: Adders and Subtractors

Asynchronous arithmetic circuits form the core of the CORDIC datapath. Designing efficient asynchronous adders and subtractors involves several considerations:

#### 6.2.1 Ripple Carry Adder (RCA) in QDI

The simplest asynchronous adder is the dual-rail ripple carry adder (RCA), which uses a cascaded chain of full adder cells. Each full adder has three inputs (\(a\), \(b\), carry-in) and produces two outputs (sum, carry-out).

**Dual-rail full adder cell:**
- Inputs: \((a_1, a_0)\), \((b_1, b_0)\), \((c_{in,1}, c_{in,0})\)
- Outputs: \((s_1, s_0)\), \((c_{out,1}, c_{out,0})\)

The full adder logic for the true output (sum = 1) is:
\[s_1 = (a_1 \oplus b_1 \oplus c_{in,1})\]

where \(\oplus\) denotes XOR. For dual-rail implementation, the false output must also be computed:
\[s_0 = (a_0 \oplus b_0 \oplus c_{in,0})\]

Actually, in dual-rail logic, we compute both:
\[s_1 = a_1 b_0 c_{in,0} + a_1 b_1 c_{in,1} + a_0 b_1 c_{in,1} + a_0 b_0 c_{in,0}\]

Wait, let me reconsider. The dual-rail XOR is more complex due to the need to represent both true and false outputs explicitly. 

The sum in binary is \(a \oplus b \oplus c\), which is true when an odd number of inputs are true. In dual-rail:
\[s_1 = (a_1 b_0 c_{in,0}) + (a_1 b_1 c_{in,0}) + (a_0 b_1 c_{in,1}) + (a_0 b_0 c_{in,1})\]
\[s_0 = (a_0 b_1 c_{in,0}) + (a_0 b_0 c_{in,0}) + (a_1 b_1 c_{in,1}) + (a_1 b_0 c_{in,1})\]

The carry-out logic is:
\[c_{out,1} = (a_1 b_1) + (a_1 c_{in,1}) + (b_1 c_{in,1})\]
\[c_{out,0} = (a_0 b_0) + (a_0 c_{in,0}) + (b_0 c_{in,0})\]

The ripple carry structure has a critical path that grows linearly with the number of bits \(n\), making it slow for wide operands. However, it remains widely used due to its simplicity and small area.

#### 6.2.2 Early Completion Adders

To improve performance, **early completion adders** detect when the result is known before all bits have been computed. For example, if all carries have stabilized to zero, the remaining bit positions will have no carry propagation, and the sum can be determined early.

Early completion detection monitors **carry generation and propagation signals**:
- **Generate (G)**: At position \(i\), \(G_i = a_i b_i\) (a carry will definitely be generated).
- **Propagate (P)**: At position \(i\), \(P_i = a_i \oplus b_i\) (a carry will propagate if it arrives from position \(i-1\)).

Early completion is signaled when the generating/propagating pattern guarantees that all remaining positions will produce stable sums.

### 6.3 Shifters and Scaling Operations

Shifters are critical components in CORDIC, as each iteration requires right-shifting by \(i\) bits. In asynchronous design, shifters can be implemented efficiently:

#### 6.3.1 Barrel Shifter

A **barrel shifter** allows shifting by any amount in a fixed amount of time. It consists of \(\log_2 n\) stages of 2:1 multiplexers, where each stage conditionally shifts by a power-of-two amount.

For a barrel shifter with \(n\)-bit input and \(m\)-bit shift amount:
- Stage \(j\) shifts by \(2^j\) bits if the \(j\)-th bit of the shift amount is 1.
- The output of stage \(j\) feeds into stage \(j+1\).

In asynchronous form, the shift amount is dual-rail encoded, and the output is valid only when all MUX stages have selected and stabilized their outputs.

#### 6.3.2 Combinational Shifting in CORDIC

In CORDIC, the shift amount is known at design time (iteration index \(i\)), not at runtime. This allows **static shifting** via wire routing: the \(i\)-th iteration simply routes the \(i\)-th bit of the shifted operand to the appropriate position. No multiplexers are required, just wire connections.

For example, to shift right by 3 bits (multiply by \(2^{-3}\)), we connect bit \(j\) of the input to bit \(j-3\) of the output (with appropriate handling of boundary cases).

### 6.4 Asynchronous Registers and Latches

Asynchronous registers store data between pipeline stages. In QDI design, registers are implemented using **asynchronous latches** with handshake control.

A typical QDI register stage consists of:
1. **Latch element**: Cross-coupled NOR or NAND gates, or an SR latch, to store one bit in dual-rail form.
2. **Control logic**: C-elements and control signals to orchestrate the four-phase handshake with the preceding and following stages.

The register becomes:
- **Transparent** (passing data through) when the write enable signal is high.
- **Opaque** (holding data) when the write enable signal is low.

For CORDIC, the register must store the current values of \((x_i, y_i, z_i)\) between iterations. The bit-width of the register determines the precision of the computation.

---

## 7. Asynchronous Control Logic and Iteration Sequencing

### 7.1 Iteration Control in Asynchronous CORDIC

Unlike synchronous CORDIC, which increments an iteration counter at each clock cycle, asynchronous CORDIC must explicitly sequence iterations. The control logic must:

1. **Detect iteration completion**: The datapath (adders, shifters, registers) completes the current iteration and asserts a completion signal.
2. **Update iteration index**: The iteration counter is incremented (or decremented, depending on the algorithm variant).
3. **Route operands for next iteration**: Multiplexers select the shifted value and the decision coefficient \(d_i\) for the next iteration.
4. **Perform handshake protocol**: Coordinate with upstream and downstream stages to manage data flow.

### 7.2 Pipelined Iteration Architecture

For higher throughput, CORDIC can be **pipelined** by replicating the datapath stages, one per iteration. In a fully pipelined asynchronous CORDIC:

1. Each iteration stage contains a complete set of adders, shifters, and registers.
2. Data flows through stages in a conveyor-belt fashion.
3. Handshake protocols between adjacent stages manage data flow, allowing multiple data tokens (corresponding to different CORDIC computations) to be in-flight simultaneously.

The advantage is that once the pipeline is full, a new result is produced every cycle (where a "cycle" is the time for one stage to complete). The disadvantage is increased area and power due to stage replication.

Asynchronous pipelining is particularly advantageous when computation times are highly variable. In synchronous design, the clock period must accommodate the slowest stage, wasting time in faster stages. In asynchronous pipelining, fast stages "pull" data from predecessors and "push" to successors without waiting for slower stages.

### 7.3 Control Using Signal Transition Graphs (STGs)

**Signal Transition Graphs (STGs)** are a formal specification language for asynchronous circuits, particularly for control logic. An STG represents the circuit behavior as a directed graph where:
- **Nodes** represent states.
- **Edges** represent signal transitions (labeled as signal+, indicating a rising transition, or signal−, indicating a falling transition).
- **Arcs are labeled** with the signal that transitions and the direction (+ for rising, − for falling).

For CORDIC iteration control, an STG might specify:
1. Wait for data valid (from preceding stage).
2. Assert request to execute current iteration.
3. Wait for iteration to complete.
4. Update iteration index.
5. Assert acknowledge to preceding stage.
6. Wait for preceding stage to de-assert request.
7. De-assert acknowledge.
8. Loop to step 1 for next iteration.

STGs provide a rigorous specification that can be checked for correctness properties (liveness, safety) before hardware implementation. Tools like **Petrify** can synthesize gate-level circuits from STGs automatically.

---

## 8. Introduction to Rajit Manohar's ACT Toolchain

### 8.1 Overview and History

The **ACT (Asynchronous Circuit Toolkit)** is a comprehensive, open-source suite of tools developed primarily by Rajit Manohar and his research groups at Caltech, Cornell University, and Yale University. ACT was designed from the ground up to support the design, synthesis, simulation, and verification of asynchronous circuits, with particular emphasis on QDI design methodologies.

The history of ACT traces back to the **MiniMIPS project** (1994–1999) at Caltech, where Manohar developed a precursor language called **CAST** (Caltech Asynchronous Synthesis Tools) to manage the complexity of designing an asynchronous microprocessor. CAST evolved through deployments at Cornell and the startup Fulcrum Microsystems, eventually becoming the open-source ACT toolkit.

ACT is **hierarchical and strongly typed**, supporting circuit description at multiple levels of abstraction:
- **Behavioral level (CHP)**: High-level specification using communicating processes.
- **Structural level (ACT)**: Hierarchical composition of modules.
- **Gate level**: Logic gate netlists.
- **Transistor level**: Low-level CMOS implementations.

### 8.2 CHP: Communicating Hardware Processes

**CHP (Communicating Hardware Processes)** is a domain-specific language within ACT that allows high-level behavioral specification of asynchronous circuits. CHP is an evolution of **Dijkstra's guarded command notation** and **Tony Hoare's Communicating Sequential Processes (CSP)**, tailored for hardware design.

#### 8.2.1 CHP Syntax and Semantics

A CHP program consists of a sequence of statements describing the behavior of a hardware process. Key CHP constructs:

**Basic Statements:**

1. **Assignment**: `x := E`
   - Evaluates expression E and assigns the result to variable x.
   - In hardware, this translates to combinational logic if E depends only on primary inputs, or to a register if E depends on variables that may change.

2. **Parallel Assignment**: `x, y := E, F`
   - Multiple assignments that occur in parallel.

3. **Send over channel**: `X!e`
   - Sends expression value e over channel X.
   - Blocking: the statement does not complete until the receiver has acknowledged receipt.

4. **Receive from channel**: `X?v`
   - Receives a value over channel X and assigns it to variable v.
   - Blocking: the statement does not complete until a sender provides data.

5. **Skip**: `skip`
   - Does nothing; used for empty branches in conditionals.

**Composition Operators:**

1. **Sequential composition**: `S; T`
   - Statement S executes, then statement T executes.
   - Similar to semicolon in traditional programming languages.

2. **Parallel composition**: `S, T`
   - Statements S and T execute concurrently (in parallel).
   - Both must complete before the next statement begins.

3. **Deterministic selection**: `[ G1 -> S1 [] G2 -> S2 ... [] Gn -> Sn ]`
   - Wait until one guard (Boolean condition) becomes true, then execute the corresponding statement.
   - The programmer asserts that at most one guard can be true simultaneously, making the choice deterministic.
   - Syntactic shorthand: `[cond]` waits for condition to become true.

4. **Non-deterministic selection**: `[| G1 -> S1 [] G2 -> S2 ... [] Gn -> Sn |]`
   - Allows multiple guards to be true; the choice is non-deterministic (requires arbiter hardware).
   - Used when mutual exclusion of guards cannot be guaranteed.

5. **Loops**: `*[ G1 -> S1 [] G2 -> S2 ... [] Gn -> Sn ]`
   - Loop body executes as long as at least one guard is true.
   - When all guards are false, the loop terminates.

6. **Syntactic replication**: `(; i : n : S[i])`
   - Generates n copies of statement S with index i ranging from 0 to n−1.

#### 8.2.2 Channels and Handshake Semantics

Channels are first-class objects in CHP, representing communication pathways between processes. A channel has a **type** and a **direction** (input or output from a process perspective).

**Channel Declaration:**
```
chan(int<8>) input_channel;
chan(bool) valid_signal;
```

**Handshake Semantics:**
- When a process executes `channel!value`, it asserts a request signal and places the value on the channel.
- The sender blocks until the receiver completes the receive operation.
- When a receiver executes `channel?var`, it waits for (and implicitly asserts) a request signal from the sender, receives the value, and acknowledges completion.
- The four-phase handshake is implicit in the blocking semantics of send and receive.

**Dual-Rail Encoding in CHP:**
While the programmer writes in single-rail logic (standard Boolean expressions), the ACT compiler automatically generates the dual-rail implementations. The type system tracks which variables are inputs/outputs and which are internal, enabling correct dual-rail synthesis.

#### 8.2.3 CHP Program Structure

A typical CHP process has the following structure:

```
process Name(chan(...) input1, output1, ...) {
  int x, y;
  int<8> counter;
  
  chp {
    *[ input1?x;       // Receive from input channel
       [ x > 0 ->      // Deterministic selection based on guard
         y := x + 1    
       [] x < 0 ->
         y := x - 1
       [] else ->
         y := 0
       ];
       output1!y       // Send to output channel
     ]
  }
}
```

This process:
1. Repeatedly receives a value from `input1`.
2. Conditionally computes y based on the value of x.
3. Sends y to `output1`.
4. Loops back to receive the next input.

### 8.3 ACT Compilation Flow

The ACT toolchain follows a hierarchical synthesis and verification flow:

```
CHP specification
       ↓
    Type checking & elaboration
       ↓
   Logic synthesis (CHP → gate-level)
       ↓
   Gate netlist
       ↓
   Timing analysis & verification
       ↓
   Place & route
       ↓
   Final layout / netlist
```

**Key stages:**

1. **Elaboration and type checking**: The ACT compiler elaborates parameterized types and checks type consistency.

2. **CHP-to-gates synthesis**: The CHP specification is translated into a gate-level netlist. This is non-trivial because the CHP semantics (particularly the blocking communication and guarded commands) must be mapped to logic gates and C-elements.

3. **Constraint extraction**: The synthesis process generates timing constraints (e.g., isochronic fork bounds, setup/hold-like relations) that must be satisfied at the physical level.

4. **Timing analysis**: A timing verification tool (e.g., **ACTtiming**) checks that the gate-level netlist satisfies all constraints derived from the CHP specification, verifying QDI correctness.

5. **Place and route**: Standard VLSI CAD tools (or custom tools within the ACT flow) perform placement and routing while respecting timing constraints.

6. **Simulation and verification**: The ACT simulator (**actsim**) can simulate both CHP and gate-level descriptions, allowing verification against reference specifications.

### 8.4 Channels and Channel-Based Communication

One of ACT's key innovations is the treatment of **channels as first-class language objects** rather than implicit wiring patterns. Channels encapsulate both data transmission and the handshake protocol, providing a level of abstraction that simplifies design and verification.

**Channel types in ACT:**

1. **Dataflow channels**: Carry data and synchronize between processes.
   - `chan(int<16>) X;` declares a channel carrying 16-bit integers.

2. **Control channels**: Carry no data, used purely for synchronization.
   - `chan(bool) ack_signal;`

3. **Arrays of channels**: Enable indexed communication (with care regarding dynamic vs. static indices).
   - `chan(int<8>) bus[4];` declares an array of four 8-bit channels.

**Two-party and multi-party channels:**

By default, channels are **two-party**: one sender and one receiver. ACT supports multi-party communication through tree or fan-out/fan-in structures, but the core two-party model is the primary use case.

### 8.5 Mapping CHP to Asynchronous Hardware

The translation of a CHP process into asynchronous hardware involves several steps:

1. **Identify control flow**: Loops, selections, and sequencing are encoded into a **state machine** using C-elements and other asynchronous control primitives.

2. **Identify datapath**: Variable assignments and expressions are implemented as combinational or sequential logic.

3. **Handshake protocol implementation**: Channel communication is realized via request/acknowledge signal pairs, coordinated by the control logic.

4. **Register insertion**: Variables that are assigned multiple times or accessed across different control states are implemented as registers (latches with handshake control).

For CORDIC, a CHP specification might have:
- An **input channel** receiving \((x, y, z)\) values for a new computation.
- A **loop** iterating over the number of CORDIC iterations.
- **Datapath operations** (addition, subtraction, shifting) performed in each iteration.
- An **output channel** sending the result after all iterations complete.

The ACT compiler synthesizes this into a hardware module with dual-rail encoded signals, C-elements for control, and arithmetic circuits for the datapath.

---

## 9. Mapping Synchronous CORDIC Algorithms to Asynchronous QDI Architecture

### 9.1 Algorithmic Mapping: From Behavioral Spec to Hardware

The transformation of a synchronous CORDIC algorithm (e.g., a C or VHDL sequential implementation) into an asynchronous QDI hardware module involves several conceptual shifts:

#### 9.1.1 Synchronous Algorithm Structure

A synchronous CORDIC implementation typically follows this pattern:

```pseudocode
function synchronous_cordic(x, y, z, iterations):
    for i in 0 to iterations-1:
        d = sign(z)
        x_new = x - d * (y >> i)
        y_new = y + d * (x >> i)
        z_new = z - d * arctan(2^(-i))
        x, y, z = x_new, y_new, z_new
    return (x, y, z)
```

Key observations:
- **Sequential iteration**: Iterations occur one after another, determined by a loop counter.
- **Register updates**: After each iteration, all three registers (x, y, z) are updated.
- **Decision logic**: The decision coefficient d is computed based on the sign of z.

#### 9.1.2 Asynchronous QDI Mapping

The asynchronous version must:

1. **Eliminate the explicit loop counter**: Instead, use a **sequence of stages** (pipelined) or a **counter register with control logic** to manage iteration count.

2. **Implement the decision logic asynchronously**: The sign of z must be computed using dual-rail logic and fed into the next operation.

3. **Express shifting and arithmetic asynchronously**: Use asynchronous adders and shifters.

4. **Manage data flow via handshakes**: Instead of synchronous register updates, use handshake protocol to pass data between stages or loop iterations.

#### 9.1.3 Iterative vs. Pipelined Implementations

**Iterative (Sequential) Asynchronous CORDIC:**
- A single datapath stage is reused for all iterations.
- A counter tracks the current iteration.
- After each iteration, the output registers feed back to the input registers (via multiplexers), and the counter is incremented.
- Latency: \(O(n)\) where n is the number of iterations (approximately equal to the bit-width).
- Area: Small, single datapath.
- Throughput: One result per n iterations; new computations start only after the previous one completes.

**Pipelined Asynchronous CORDIC:**
- Each iteration is implemented as a separate stage in a pipeline.
- Data flows through stages sequentially, with handshaking between stages.
- Multiple computations can be in-flight simultaneously.
- Latency: \(O(n)\) stages, but in asynchronous systems, latency depends on actual computation times, not worst-case.
- Area: Large, due to stage replication.
- Throughput: One result per iteration (once pipeline is full), much higher than iterative approach.

For an asynchronous system, pipelined CORDIC is often preferred because:
- Asynchronous handshaking naturally adapts to variable stage delays.
- If some stages are faster, they process data more frequently, while slower stages auto-throttle.
- The throughput is higher, better matching typical signal processing rates.

### 9.2 CHP Specification of Asynchronous CORDIC

A CHP specification for pipelined asynchronous CORDIC might have the following structure:

#### 9.2.1 Top-Level Process

```
process CORDIC_Module(
    chan(VECTOR_TYPE) input_port,    // Input: (x, y, z)
    chan(VECTOR_TYPE) output_port    // Output: (x, y, z)
) {
    int<PRECISION> x, y, z;
    
    chp {
        *[ 
            input_port?x, y, z;        // Receive input vector
            // Stages 0 to N-1 process the vector
            // (Detail follows below)
            output_port!x, y, z        // Send result
        ]
    }
}
```

#### 9.2.2 Individual Pipeline Stage (Example: Stage i)

```
process CORDIC_Stage(
    int stage_index,                 // Fixed: e.g., 0, 1, 2, ...
    chan(VECTOR_TYPE) in,           // Input from previous stage
    chan(VECTOR_TYPE) out,          // Output to next stage
) {
    int<PRECISION> x_in, y_in, z_in;
    int<PRECISION> x_out, y_out, z_out;
    bool d;  // Decision coefficient
    
    chp {
        *[
            in?x_in, y_in, z_in;
            
            // Compute decision coefficient based on sign of z
            [ z_in >= 0 ->
                d := true
            [] else ->
                d := false
            ];
            
            // Conditional rotation
            [ d ->
                // Rotate counter-clockwise (d = +1)
                x_out := x_in - (y_in >> stage_index);
                y_out := y_in + (x_in >> stage_index);
                z_out := z_in - ARCTAN_TABLE[stage_index]
            [] else ->
                // Rotate clockwise (d = -1)
                x_out := x_in + (y_in >> stage_index);
                y_out := y_in - (x_in >> stage_index);
                z_out := z_in + ARCTAN_TABLE[stage_index]
            ];
            
            out!x_out, y_out, z_out
        ]
    }
}
```

**Key features:**

1. **Guarded conditional**: The `[ condition -> ... [] else -> ... ]` structure handles the decision logic asynchronously.

2. **Channels for pipeline communication**: Each stage receives from the previous stage and sends to the next via channels.

3. **Shift operations**: Represented as `>> stage_index`, which the compiler translates to wiring (no multiplexers).

4. **ARCTAN_TABLE**: Pre-computed constants \(\arctan(2^{-i})\) stored as data.

#### 9.2.3 Multi-Stage Composition

A full pipelined CORDIC is assembled by chaining stages:

```
process CORDIC_Pipeline(
    chan(VECTOR_TYPE) input_port,
    chan(VECTOR_TYPE) output_port
) {
    chan(VECTOR_TYPE) stage_connections[NUM_STAGES + 1];
    
    // Instantiate all pipeline stages
    CORDIC_Stage(0, input_port, stage_connections[1]);
    CORDIC_Stage(1, stage_connections[1], stage_connections[2]);
    ...
    CORDIC_Stage(NUM_STAGES - 1, 
                 stage_connections[NUM_STAGES], 
                 output_port);
}
```

The hierarchical composition of processes allows the pipeline to be described at a high level, with the ACT compiler handling the detailed synthesis of handshake logic.

### 9.3 Design Space Exploration in CHP

The CHP specification enables easy exploration of design trade-offs:

1. **Iteration count**: By changing `NUM_STAGES`, you select the precision and convergence behavior.

2. **Precision**: Fixed-point format (number of integer and fractional bits) can be parameterized.

3. **Iteration vs. pipeline**: Switching between iterative and pipelined implementations requires different process structures but the same CHP language.

4. **Conditional vs. arithmetic shifting**: Some designs precompute all possible shifted values and use multiplexers; others use combinational shifters. CHP abstracts this choice.

---

## 10. Verification Strategies for Asynchronous Hardware

### 10.1 Correctness Challenges in Asynchronous Design

Verification of asynchronous circuits presents unique challenges:

1. **Exponential state space**: Without a clock to discretize time, the potential state space is much larger, making exhaustive simulation challenging.

2. **Timing-dependent behavior**: The circuit's behavior depends on actual delays, which are implementation-specific. A circuit might be correct for one technology but incorrect for another due to timing changes.

3. **Metastability**: Asynchronous designs must carefully manage metastability, particularly at clock domain crossings or in arbiters.

4. **Formal verification complexity**: Proving QDI correctness requires formal reasoning about delay-insensitive behavior, which is more complex than proving synchronous correctness.

### 10.2 ACT Verification Tools

The ACT toolchain includes several verification mechanisms:

#### 10.2.1 CHP-Level Simulation

The **actsim** simulator allows simulation of CHP processes against test benches:

```
process testbench {
    chan(VECTOR_TYPE) input_stim;
    chan(VECTOR_TYPE) output_resp;
    
    chp {
        // Generate test stimuli
        input_stim!(x1, y1, z1);
        output_resp?x_out, y_out, z_out;
        
        // Check correctness (compare against reference)
        [ ABS(x_out - expected_x) < TOLERANCE ->
            log("Test passed")
        [] else ->
            warn("Test failed")
        ]
    }
}
```

The simulator executes CHP-level code, allowing verification before hardware synthesis. Test coverage can be systematically explored.

#### 10.2.2 Gate-Level Verification

After synthesis, gate-level descriptions can be simulated using standard simulators (e.g., Verilog simulators) to verify that the synthesized implementation matches the CHP specification.

#### 10.2.3 Timing Analysis and Constraint Checking

The **ACTtiming** tool and related timing verifiers check that gate-level netlists satisfy all timing constraints derived from the CHP specification. This includes:

- **Isochronic fork bounds**: Verifies that all forks of a signal have balanced delays.
- **Setup/hold-like timing**: Ensures that handshake signals maintain proper temporal relationships.
- **C-element delay assumptions**: Verifies that internal feedback is faster than external paths.

### 10.3 Formal Verification of QDI Correctness

For formal assurance of QDI correctness, several approaches exist:

#### 10.3.1 Model Checking

**Model checking** involves constructing a finite (or abstracted) state machine representing the circuit and checking that all reachable states satisfy a given specification (expressed in temporal logic, e.g., LTL or CTL).

For asynchronous circuits, models must account for:
- **All possible interleavings** of concurrent operations.
- **Non-deterministic choice** in arbiters or when guards are mutually exclusive but both satisfied.
- **Delay variation** (in bounded or unbounded models).

Tools like **NuSMV** and **UPPAAL** can be used, though they often require manual model abstraction due to state space explosion.

#### 10.3.2 Theorem Proving

**Theorem proving** (e.g., using Coq, Isabelle, or ACL2) allows machine-checked proofs of circuit correctness. For CORDIC:

1. **Algorithmic correctness**: Prove that the CORDIC iteration sequence converges to the desired angle and produces correct sine/cosine values.

2. **Fixed-point arithmetic correctness**: Prove bounds on quantization error and overall accuracy.

3. **Asynchronous implementation correctness**: Prove that the asynchronous circuit implementation correctly executes the CORDIC algorithm despite unbounded delays.

Formal proofs provide the highest assurance but require significant effort to develop.

#### 10.3.3 Equivalence Checking

**Equivalence checking** verifies that the synthesized gate-level design is equivalent to a higher-level reference specification (e.g., the CHP specification). Tools like **Symplify** and others can check combinational and sequential equivalence.

For asynchronous circuits, equivalence checking must account for dual-rail encoding and handshake behavior, making it more complex than synchronous equivalence checking.

### 10.4 Testing Strategies

**Simulation-based testing** remains crucial:

1. **Unit testing**: Verify individual components (adders, shifters, CORDIC stages) against known correct results.

2. **Integration testing**: Verify pipelined CORDIC against reference implementations (e.g., standard C/Python CORDIC implementations or floating-point calculations).

3. **Corner case testing**: Test boundary conditions, such as:
   - Zero input vectors.
   - Maximum magnitude inputs.
   - Very small angles.
   - Angles near zero and near the convergence boundary.

4. **Quantization error analysis**: Verify that fixed-point quantization does not cause unacceptable errors. Compare outputs against high-precision reference computations.

5. **Timing robustness testing**: In gate-level simulation, vary delays and verify correct operation across process corners and delay ranges.

---

## 11. Assumptions and Limitations

### 11.1 Stated Design Assumptions

1. **Isochronic fork assumption**: It is assumed that signal forks, particularly in control logic and at C-element inputs, can be balanced to maintain the isochronic fork assumption. Placement and routing tools must enforce this.

2. **Finite, positive delays**: All gate and wire delays are assumed to be positive (causality) and finite. Unbounded delay is tolerated, but the circuit must eventually stabilize.

3. **No metastability**: It is assumed that arbiters (used in non-deterministic selections) are properly designed to avoid metastability, or that the probability of metastability is acceptably low.

4. **Correct channel abstraction**: CHP channels abstract handshaking details. The ACT compiler is assumed to correctly synthesize these channels into correct handshake logic.

5. **Dual-rail implementation fidelity**: Dual-rail logic synthesis is assumed to produce correct logic that properly separates true and false computations.

6. **Fixed-point arithmetic**: It is assumed that the chosen fixed-point precision (number of bits) is sufficient for the target application. Overflow and underflow are assumed not to occur, or to be handled by the application.

### 11.2 Limitations

1. **Area and power overhead**: Dual-rail encoding doubles the number of wires per logical signal and adds control overhead (C-elements, handshake logic). This increases area and power compared to synchronous single-rail implementations.

2. **Precision limitations**: Fixed-point arithmetic has limited precision compared to floating-point. For applications requiring dynamic range or very high accuracy, floating-point or higher bit-widths may be necessary.

3. **Convergence guarantee**: CORDIC convergence is guaranteed only for inputs within the convergence range (\(\approx 99.9°\) for circular rotation mode). Inputs outside this range require preprocessing (quadrant adjustment).

4. **Complexity of formal verification**: Formal verification of large circuits is computationally intensive. For sizable CORDIC designs, exhaustive formal proof may be impractical; statistical testing and simulation-based verification become necessary.

5. **Throughput-latency trade-off**: Pipelined designs have lower latency but higher area. Iterative designs have smaller area but higher latency and lower throughput. The choice depends on application requirements.

6. **Temperature and voltage sensitivity**: While QDI design is robust to PVT variations at the gate level, actual silicon characteristics (leakage currents, process corners) can still affect performance and must be characterized.

---

## 12. Future Extensions and Advanced Topics

### 12.1 Hybrid CORDIC and Angle Recoding

**Hybrid CORDIC** combines an iterative first stage (computing a few iterations sequentially) with a pipelined second stage (executing remaining iterations in parallel). This approach balances area and latency.

**Angle recoding** methods (e.g., Backward Angle Recoding, BAR) reduce the number of required iterations by selecting a non-sequential subset of micro-angles that sum to the target angle more efficiently. This requires:

1. Computing the recoding offline (in software).
2. Adjusting the iteration count and angle lookup table.
3. Careful handling of convergence guarantees with reduced iterations.

Extension of ACT CORDIC to support angle recoding would involve:
- Parameterizing the angle sequence and lookup table.
- Dynamically determining iteration count based on recoding.
- Synthesis challenges in handling variable-length computations.

### 12.2 Subthreshold and Ultra-Low-Power Operation

QDI circuits have shown promise in **subthreshold (near-threshold) operation**, where circuits operate at supply voltages far below nominal levels, reducing power consumption to the ultra-low range required for battery-less IoT and RFID applications.

For asynchronous CORDIC in subthreshold regime:
- Delays increase dramatically (exponentially with voltage), but circuit operation remains correct due to delay-insensitive design.
- Power consumption decreases exponentially, enabling micro-watt-range operation.
- This is a key advantage over synchronous designs, which face severe timing closure challenges at subthreshold.

### 12.3 Reconfigurable Asynchronous CORDIC

A **reconfigurable CORDIC** module could support:
- Selection between rotation and vectoring modes (via parameterized control logic).
- Selection between circular, hyperbolic, and linear coordinate systems.
- Programmable iteration count and precision.

In ACT, this would involve:
- Parameterized CHP processes with mode/configuration inputs.
- Conditional compilation or dynamic mode selection.
- Careful management of control complexity to avoid excessive overhead.

### 12.4 Integration with Asynchronous Signal Processing Pipelines

For signal processing applications, asynchronous CORDIC can be integrated into larger asynchronous processing pipelines (e.g., FFT processors, correlators, beamformers). The benefits include:

- **Elastic pipelines**: Natural data-driven flow without clock gating.
- **Reduced power**: Only active stages consume power.
- **Modular design**: CORDIC can be plugged into various applications.

Challenges include:
- Managing data rates and buffering between asynchronous modules.
- Synchronization with external (synchronous) components if needed.
- Verification of the end-to-end pipeline.

### 12.5 Neuromorphic and Event-Driven Applications

Asynchronous CORDIC is naturally suited to **neuromorphic computing** and event-driven sensory processing, where spike timing (not clock cycles) drives computation. CORDIC can compute rotation and magnitude directly from spike-encoded inputs, enabling low-power neuromorphic signal processing.

---

## 13. Conclusion

This document has presented a comprehensive theoretical foundation for the design and verification of an asynchronous CORDIC hardware module using the ACT toolchain. The work integrates three major components:

1. **CORDIC Algorithm**: A mathematically elegant and computationally efficient method for computing transcendental functions via shift-and-add operations. Its iterative structure and data-dependent convergence make it well-suited to asynchronous implementation.

2. **Asynchronous QDI Design**: A robust methodology for designing circuits that operate correctly despite unbounded delays, using dual-rail encoding, handshake protocols, C-elements, and isochronic fork assumptions. QDI provides a pragmatic balance between theoretical ideals (fully delay-insensitive) and practical constraints (area, power, complexity).

3. **ACT Toolchain and CHP**: A modern, open-source framework for high-level behavioral specification (CHP), automatic synthesis to gate-level (dual-rail logic), timing analysis, and verification. CHP raises the level of abstraction, allowing designers to focus on algorithms and control flow rather than low-level handshaking details.

The integration of these three elements enables the design of **efficient, correct, and verifiable asynchronous CORDIC hardware** suitable for applications requiring transcendental function computation with low power, robustness to process variations, and natural adaptation to data-dependent computation times.

Key theoretical contributions of this document:

- **Rigorous mathematical treatment** of CORDIC convergence, fixed-point arithmetic, and function computation.
- **Detailed explanation of QDI design principles**, including dual-rail encoding, C-element behavior, and the isochronic fork assumption.
- **Comprehensive overview of the ACT toolchain**, with emphasis on CHP specification and channel-based communication.
- **Mapping procedures** for transforming synchronous CORDIC algorithms into asynchronous QDI architectures.
- **Verification strategies** spanning CHP simulation, gate-level checking, timing analysis, and formal methods.

The design of the asynchronous CORDIC module, when undertaken with this theoretical foundation, will be rigorous, well-justified, and trustworthy for senior academic and professional review.

