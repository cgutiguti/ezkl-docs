---
icon: workflow
order: 7
---
#### verifying with the EVM ◊

Verification can also be run with an EVM verifier. This can be done by generating a verifier smart contract after performing setup.

```bash
# Create new SRS
ezkl gen-srs --logrows 17 --params-path=17.srs
```

```bash
# Create new circuit parameters
ezkl gen-circuit-params --calibration-target resources --model examples/onnx/1l_relu/network.onnx --circuit-params-path circuit.json
```

```bash
# Set up a new circuit
ezkl setup -M examples/onnx/1l_relu/network.onnx --params-path=17.srs --vk-path=vk.key --pk-path=pk.key --circuit-params-path=circuit.json
```

```bash
# gen evm verifier
ezkl create-evm-verifier --deployment-code-path 1l_relu.code --params-path=17.srs --vk-path vk.key --sol-code-path 1l_relu.sol --circuit-params-path=circuit.json
```

```bash
ezkl prove --transcript=evm -D ./examples/onnx/1l_relu/input.json -M ./examples/onnx/1l_relu/network.onnx --proof-path 1l_relu.pf --pk-path pk.key --params-path=17.srs --circuit-params-path=circuit.json 
```

```bash
# Verify (EVM)
ezkl verify-evm --proof-path 1l_relu.pf --deployment-code-path 1l_relu.code
```

Note that the `.sol` file above can be deployed and composed with other Solidity contracts, via a `verify()` function. Please read [this document](https://hackmd.io/QOHOPeryRsOraO7FUnG-tg) for more information about the interface of the contract, how to obtain the data needed for its function parameters, and its limitations.

The above pipeline can also be run using proof aggregation to reduce the final proof size and the size and execution cost of the on-chain verifier. A sample pipeline for doing so would be:

```bash
# Generate a new SRS. We use 20 since aggregation requires larger circuits (more commonly 23+).
ezkl gen-srs --logrows 20 --params-path=20.srs
```

```bash
# Create new circuit parameters
ezkl gen-circuit-params --calibration-target resources --model examples/onnx/1l_relu/network.onnx --circuit-params-path circuit.json
```

```bash
# Set up a new circuit
ezkl setup  -M examples/onnx/1l_relu/network.onnx --params-path=20.srs --vk-path=vk.key --pk-path=pk.key --circuit-params-path=circuit.json
```

```bash
# Single proof -> single proof we are going to feed into aggregation circuit. (Mock)-verifies + verifies natively as sanity check
ezkl prove --transcript=poseidon --strategy=accum -D ./examples/onnx/1l_relu/input.json -M ./examples/onnx/1l_relu/network.onnx --proof-path 1l_relu.pf --params-path=20.srs  --pk-path=pk.key --circuit-params-path=circuit.json
```

```bash
# Aggregate -> generates aggregate proof and also (mock)-verifies + verifies natively as sanity check
ezkl aggregate --logrows=20 --aggregation-snarks=1l_relu.pf --aggregation-vk-paths vk.key --vk-path aggr_1l_relu.vk --proof-path aggr_1l_relu.pf --params-path=20.srs --circuit-params-paths=circuit.json
```

```bash
# Generate verifier code -> create the EVM verifier code
ezkl create-evm-verifier-aggr --deployment-code-path aggr_1l_relu.code --params-path=20.srs --vk-path aggr_1l_relu.vk
```

```bash
# Verify (EVM) ->
ezkl verify-aggr --logrows=20 --proof-path aggr_1l_relu.pf --params-path=20.srs --vk-path aggr_1l_relu.vk
```

Also note that this may require a local [solc](https://docs.soliditylang.org/en/v0.8.17/installing-solidity.html) installation, and that aggregated proof verification in Solidity is not currently supported. You can follow the SolidityLang instructions linked above, or you can use [svm-rs](https://github.com/alloy-rs/svm-rs) to install solc. Here's how:

Install svm-rs:
```bash
cargo install svm-rs
```

Install a recent Solidity version (we use 0.8.20 in our implementation):
```bash
svm install 0.8.20
```

Verify your Solidity version:
```bash
solc --version
```
