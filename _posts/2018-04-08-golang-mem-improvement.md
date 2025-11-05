---
layout: post
title: "golang内存优化"
date: 2018-04-28
tags: Golang
---

<table>
    <thead>
        <th>原数据结构</th>
        <th>优化数据结构</th>
        <th>优化点</th>
    </thead>
    <tbody>
        <tr>
            <td>map[string]SampleStruct</td>
            <td>map[[32]byte]SampleStruct</td>
            <td>key使用值类型避免对map遍历</td>
        </tr>
        <tr>
            <td>map[int]*SampleStruct</td>
            <td>map[int]SampleStruct</td>
            <td>Value使用值类型避免对map遍历</td>
        </tr>
        <tr>
            <td>sampleSlice []float64</td>
            <td>sampleSlice [32]float64</td>
            <td>利用值类型代替对象类型</td>       
        </tr>
    </tbody>
</table>

