# SPDX-License-Identifier: BSD-3-Clause
# Copyright (c) 2020-2022 Intel Corporation

sources = files('main.c', 'parse-args.c', 'stats.c', 'acl-func.c')

deps += [
    acl,
    bpf_dep,
    events,
    mempool,
    metrics,
    mmap,
    net,
    pktdev,
    pktmbuf,
    pmd_af_xdp,
    pmd_af_packet,
    pmd_memif,
    ring,
    txbuff,
    uds,
    udsc,
    xskdev,
]

executable('cndpfwd',
	sources,
	c_args: cflags,
	install: true,
	dependencies: deps)