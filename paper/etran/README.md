Important caveat: I was unable to directly access the source code from the eTran-NSDI25/eTran-linux kernel repository (it exists on GitHub but isn't well-indexed for search). The analysis is therefore reconstructed from the paper's detailed descriptions in §3.2.1, §4.1, §4.2, §5, and Appendix C, cross-referenced with the standard Linux v6.6 XDP/AF_XDP codebase. Structures marked with ★ are inferred.
Key findings:

XDP_GEN is placed in xdp_do_flush(), called at the end of every NAPI poll — it runs a budget-bounded loop over pre-allocated xdp_frame pools from page_pool
It reuses the struct xdp_md context from standard XDP, but exposes a restricted subset of eBPF helpers (notably excluding bpf_redirect_map to prevent arbitrary packet redirection)
The coordination pattern is: XDP (ingress) pushes ACK/credit metadata into a per-CPU eBPF queue → XDP_GEN pops it and constructs actual packets in pre-allocated frames → transmits via XDP_TX
For Homa, XDP_GEN is significantly more complex, requiring bpf_tail_call to break the credit scheduling logic across multiple programs to satisfy verifier instruction limits
