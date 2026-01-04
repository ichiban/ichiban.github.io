+++
date = '2025-12-30T16:24:38+09:00'
title = 'Structure of Arrays in Go'
tags = ['Go', 'data structure']
+++

# AoS vs SoA

Let's say we're modeling 3D particles.
We define a struct for the particle with its position, velocity, and mass.


```go
type Particle struct {
	X, Y, Z    float32
	VX, VY, VZ float32
	Mass       float32
}
```

When we have many particles and update only a subset of fields, a plain []Particle can be less cache-efficient because each iteration still loads the entire struct (including fields we don’t use in that loop).

```goat

.----+----+----+--------------------------+
|   X|   X|   X|                          |
+----+----+----+--------------------------+
|   Y|   Y|   Y|                          |
+----+----+----+--------------------------+
|   Z|   Z|   Z|                          |
+----+----+----+--------------------------+
|  VX|  VX|  VX|                          |
+----+----+----+--------------------------+
|  VY|  VY|  VY|                          |
+----+----+----+--------------------------+
|  VZ|  VZ|  VZ|                          |
+----+----+----+--------------------------+
|Mass|Mass|Mass|                          |
+----+----+----+--------------------------+
|    |    |    |                          |
|    |    |    |                          |
'----+----+----+--------------------------+

```

Instead of the typical `[]Particle` (an Array of Structures, or AoS), we can model a bunch of particles with a `ParticleSlice` struct that has the same fields as Particle, but where each field is a slice.


```go
type ParticleSlice struct {
	X, Y, Z    []float32
	VX, VY, VZ []float32
	Mass       []float32
}
```

With this layout—Structure of Arrays (SoA)—we touch only the fields we need in a tight loop and can ignore the rest.


```goat

.----+---.                           .-+-+-+------.
|   X| *-+-------------------------->| | | | ...  |
+----+---+                           '-+-+-+------'
|   Y| *-+------------------------.                
+----+---+                        |  .-+-+-+------.
|   Z| *-+----------------------. '->| | | | ...  |
+----+---+                      |    '-+-+-+------'
|  VX| *-+-------------------.  |                  
+----+---+                   |  |    .-+-+-+------.
|  VY| *-+----------------.  |  '--->| | | | ...  |
+----+---+                |  |       '-+-+-+------'
|  VZ| *-+-------------.  |  |                     
+----+---+             |  |  |       .-+-+-+------.
|Mass| *-+----------.  |  |  '------>| | | | ...  |
+----+---+          |  |  |          '-+-+-+------'
|    |   |          |  |  |                        
|    |   |          |  |  |          .-+-+-+------.
'----+---'          |  |  '--------->| | | | ...  |
                    |  |             '-+-+-+------'
                    |  |
                    |  |             .-+-+-+------.
                    |  '------------>| | | | ...  |
                    |                '-+-+-+------'
                    |
                    |                .-+-+-+------.
                    '--------------->| | | | ...  |
                                     '-+-+-+------'
```



# Go support for SoA

Does Go support SoA out of the box? Not directly. There was [a proposal](https://github.com/golang/go/issues/64926), but it was closed as not planned. In that issue, [ajwerner suggested that it could be achieved with a (hypothetical) code generator](https://github.com/golang/go/issues/64926#issuecomment-1875789572).

So, [I made one](https://github.com/ichiban/soa).


# The code generator and library

`github.com/ichiban/soa/cmd/soagen` is a code generator you can install as a Go tool (Go 1.24+):

```console
go get -tool github.com/ichiban/soa/cmd/soagen
```

Then `go tool soagen [the file that defines Particle]` generates a Go file with `ParticleSlice`.

Because `ParticleSlice` is not a single slice value, you can't manipulate it with `slices` directly. Instead, you can use `github.com/ichiban/soa` for helper APIs that mirror common slice operations.

# Benchmark

Here's a simple benchmark that shows a modest difference on my machine:

```go
func BenchmarkGravity(b *testing.B) {
	const dt = float32(0.01)
	const gravity = float32(-9.8)
	const numParticles = 1_000_000

	particles := make([]Particle, numParticles)
	for i := range particles {
		particles[i] = Particle{
			X:    rand.Float32(),
			Y:    rand.Float32(),
			Z:    rand.Float32(),
			VX:   rand.Float32(),
			VY:   rand.Float32(),
			VZ:   rand.Float32(),
			Mass: rand.Float32(),
		}
	}

	particlesSoA := Make[ParticleSlice](numParticles, numParticles)
	for i := 0; i < numParticles; i++ {
		particlesSoA.Set(i, Particle{
			X:    rand.Float32(),
			Y:    rand.Float32(),
			Z:    rand.Float32(),
			VX:   rand.Float32(),
			VY:   rand.Float32(),
			VZ:   rand.Float32(),
			Mass: rand.Float32(),
		})
	}

	b.Run("array of structures", func(b *testing.B) {
		for n := 0; n < b.N; n++ {
			for i := 0; i < numParticles; i++ {
				vy := particles[i].VY + gravity*dt
				particles[i].VY = vy
				particles[i].Y += vy * dt
			}
		}
	})

	b.Run("structure of arrays", func(b *testing.B) {
		for n := 0; n < b.N; n++ {
			for i := 0; i < numParticles; i++ {
				vy := particlesSoA.VY[i] + gravity*dt
				particlesSoA.VY[i] = vy
				particlesSoA.Y[i] += vy * dt
			}
		}
	})
}
```

```console
$ go test -bench github.com/ichiban/soa -bench BenchmarkGravity
goos: darwin
goarch: arm64
pkg: github.com/ichiban/soa
cpu: Apple M1
BenchmarkGravity/array_of_structures-8              1080           1219608 ns/op
BenchmarkGravity/structure_of_arrays-8              1159           1026934 ns/op
PASS
ok      github.com/ichiban/soa  3.974s
```

https://www.reddit.com/r/golang/comments/1jeyhax/soa_structure_of_arrays_in_go/
