<script>
	import { onMount } from 'svelte';
	import { screenType } from '$lib/store/store';
	import { page } from '$app/stores';
	import { afterNavigate } from '$app/navigation';

	import * as THREE from 'three';

	// import vertexShader from './shaders/perlin/perlinVertex.glsl';
	// import fragmentShader from './shaders/perlin/perlinFragment.glsl';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { ImprovedNoise } from 'three/addons/math/ImprovedNoise.js';

	import { GUI } from 'three/addons/libs/lil-gui.module.min.js';
	import WebGL from 'three/addons/capabilities/WebGL.js';

	if ( WebGL.isWebGL2Available() === false ) {
		document.body.appendChild( WebGL.getWebGL2ErrorMessage() );
	}

	let container;
  let controls;

	let camera, scene, renderer;
	let mesh;

	let width = window.innerWidth;
	let height = window.innerHeight;

  // ---------------------------------------------------------------------------
	// Texture -------------------------------------------------------------------
  // ---------------------------------------------------------------------------

	const size = 128;
	const data = new Uint8Array( size * size * size );

	let i = 0;
	const perlin = new ImprovedNoise();
	const vector = new THREE.Vector3();

	for ( let z = 0; z < size; z ++ ) {

		for ( let y = 0; y < size; y ++ ) {

			for ( let x = 0; x < size; x ++ ) {

				vector.set( x, y, z ).divideScalar( size );

				const d = perlin.noise( vector.x * 6.5, vector.y * 6.5, vector.z * 6.5 );

				data[ i ++ ] = d * 128 + 128;

			}

		}

	}

	const texture = new THREE.Data3DTexture( data, size, size, size );
	texture.format = THREE.RedFormat;
	texture.minFilter = THREE.LinearFilter;
	texture.magFilter = THREE.LinearFilter;
	texture.unpackAlignment = 1;
	texture.needsUpdate = true;

	// ---------------------------------------------------------------------------
	// Planes --------------------------------------------------------------------
  // ---------------------------------------------------------------------------

	// Add these to the top, near other variables
	let planes = [];
	const planeHelpers = [];
	const planeMatrices = [];


  // ---------------------------------------------------------------------------
	// Material ------------------------------------------------------------------
  // ---------------------------------------------------------------------------

	const vertexShader = /* glsl */`
		in vec3 position;

		uniform mat4 modelMatrix;
		uniform mat4 modelViewMatrix;
		uniform mat4 projectionMatrix;
		uniform vec3 cameraPos;

		out vec3 vOrigin;
		out vec3 vDirection;

		void main() {
			vec4 mvPosition = modelViewMatrix * vec4( position, 1.0 );

			vOrigin = vec3( inverse( modelMatrix ) * vec4( cameraPos, 1.0 ) ).xyz;
			vDirection = position - vOrigin;

			gl_Position = projectionMatrix * mvPosition;
		}
	`;

	const fragmentShader = /* glsl */`
		precision highp float;
		precision highp sampler3D;

		uniform mat4 modelViewMatrix;
		uniform mat4 projectionMatrix;

		in vec3 vOrigin;
		in vec3 vDirection;

		out vec4 color;

		uniform sampler3D map;

		uniform float threshold;
		uniform float steps;
		uniform float baseOpacity;

		vec2 hitBox( vec3 orig, vec3 dir ) {
			const vec3 box_min = vec3( - 0.5 );
			const vec3 box_max = vec3( 0.5 );
			vec3 inv_dir = 1.0 / dir;
			vec3 tmin_tmp = ( box_min - orig ) * inv_dir;
			vec3 tmax_tmp = ( box_max - orig ) * inv_dir;
			vec3 tmin = min( tmin_tmp, tmax_tmp );
			vec3 tmax = max( tmin_tmp, tmax_tmp );
			float t0 = max( tmin.x, max( tmin.y, tmin.z ) );
			float t1 = min( tmax.x, min( tmax.y, tmax.z ) );
			return vec2( t0, t1 );
		}

		float sample1( vec3 p ) {
			return texture( map, p ).r;
		}

		#define epsilon .001

		vec3 normal( vec3 coord ) {
			if ( coord.x < epsilon ) return vec3( 1.0, 0.0, 0.0 );
			if ( coord.y < epsilon ) return vec3( 0.0, 1.0, 0.0 );
			if ( coord.z < epsilon ) return vec3( 0.0, 0.0, 1.0 );
			if ( coord.x > 1.0 - epsilon ) return vec3( - 1.0, 0.0, 0.0 );
			if ( coord.y > 1.0 - epsilon ) return vec3( 0.0, - 1.0, 0.0 );
			if ( coord.z > 1.0 - epsilon ) return vec3( 0.0, 0.0, - 1.0 );

			float step = 0.01;
			float x = sample1( coord + vec3( - step, 0.0, 0.0 ) ) - sample1( coord + vec3( step, 0.0, 0.0 ) );
			float y = sample1( coord + vec3( 0.0, - step, 0.0 ) ) - sample1( coord + vec3( 0.0, step, 0.0 ) );
			float z = sample1( coord + vec3( 0.0, 0.0, - step ) ) - sample1( coord + vec3( 0.0, 0.0, step ) );

			return normalize( vec3( x, y, z ) );
		}

		void main(){

			color = vec4(0.0);
			vec3 rayDir = normalize( vDirection );
			vec2 bounds = hitBox( vOrigin, rayDir );

			if ( bounds.x > bounds.y ) discard;

			bounds.x = max( bounds.x, 0.0 );

			vec3 p = vOrigin + bounds.x * rayDir;
			vec3 inc = 1.0 / abs( rayDir );
			float delta = min( inc.x, min( inc.y, inc.z ) );
			delta /= steps;

			for ( float t = bounds.x; t < bounds.y; t += delta ) {

				float d = sample1( p + 0.5 );

				if ( d > threshold ) {

					color.rgb = normal( p + 0.5 ) * 0.5 + ( p * 1.5 + 0.25 );
					color.a = baseOpacity;
					break;

				}

				p += rayDir * delta;

			}

			if ( color.a == 0.0 ) discard;

		}
	`;

	// Plane shaders
	const planeVertexShader = /* glsl */`
		in vec3 position;
		in vec2 uv;

		uniform mat4 modelViewMatrix;
		uniform mat4 projectionMatrix;

		out vec2 vUv;

		void main() {
			vUv = uv;
			gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
		}
	`;

	const planeFragmentShader = /* glsl */`
    precision highp float;
    precision highp sampler3D;

    in vec2 vUv;

    out vec4 color;

    uniform sampler3D map;
    uniform float slicePosition;
		uniform float threshold;
    uniform int sliceAxis; // 0 = X, 1 = Y, 2 = Z

    void main() {
        vec3 samplePos;

        if (sliceAxis == 0) {
            samplePos = vec3(slicePosition, vUv.y, 1.0 - vUv.x); // something is inverted here
        } else if (sliceAxis == 1) {
            samplePos = vec3(vUv.x, slicePosition, vUv.y);
        } else {
            samplePos = vec3(vUv.x, vUv.y, slicePosition);
        }

        float sampledValue = texture(map, samplePos).r; // Renamed variable
        color  = vec4(vec3(sampledValue), 1.0); // Always opacity 1

				if ( color.x < threshold ) color = vec4(0.137,0.137,0.137,1.0);
				if ( color.y < threshold ) color = vec4(0.137,0.137,0.137,1.0);
				if ( color.z < threshold ) color = vec4(0.137,0.137,0.137,1.0);

				// Check if the pixel is at the edge and set its color to white
        float edgeThreshold = 0.001; // Small value to account for potential inaccuracies
        if(vUv.x < edgeThreshold || vUv.x > 1.0 - edgeThreshold || vUv.y < edgeThreshold || vUv.y > 1.0 - edgeThreshold) {
            color = vec4(0.5, 0.5, 0.5, 1.0); // Set color to white
        }
    }
`;




	init();
	animate();

	function init() {
		camera = new THREE.PerspectiveCamera(20, width / height, 0.1, 10000);
		camera.position.x = 2; 	
		camera.position.y = 2; 	
		camera.position.z = 4; 	

		scene = new THREE.Scene();
		scene.background = new THREE.Color(0x232323);

		setScene();

		renderer = new THREE.WebGLRenderer({ antialias: false });
		renderer.setPixelRatio(window.devicePixelRatio);
		renderer.setSize(width, height);

		renderer.setClearColor(0x232323, 0);
		renderer.autoClearColor = true;
		renderer.autoClearDepth = true;
		renderer.autoClearStencil = true;

		controls = new OrbitControls(camera, renderer.domElement);
		controls.target.set(0, 0, 0);  // Explicitly set target to origin
		controls.update();

		onMount(() => {
			container.appendChild(renderer.domElement);
		});
	}

	function setHome () {
		const geometry = new THREE.BoxGeometry( 1, 1, 1 );
			const material = new THREE.RawShaderMaterial( {
				glslVersion: THREE.GLSL3,
				uniforms: {
					map: { value: texture },
					cameraPos: { value: new THREE.Vector3() },
					threshold: { value: 0.5 },
					steps: { value: 200 },
					baseOpacity: { value: 0.5 },
				},
				vertexShader,
				fragmentShader,
				side: THREE.BackSide,
				transparent: true,
			} );

			mesh = new THREE.Mesh( geometry, material );
			scene.add( mesh );

			// Add box helper
			const box = new THREE.BoxHelper( mesh, 0x808080);
			scene.add( box );

			for (let i = 0; i < 3; i++) {
				const planeGeom = new THREE.PlaneGeometry(1, 1);
				const planeMat = new THREE.RawShaderMaterial({
						glslVersion: THREE.GLSL3,
						uniforms: {
								map: { value: texture },
								cameraPos: { value: new THREE.Vector3() },
								threshold: { value: 0.5 },
								steps: { value: 200 },
								baseOpacity: { value: 1.0 },
								slicePosition: { value: 0.5 },
								sliceAxis: { value: null },
						},
						vertexShader: planeVertexShader,
						fragmentShader: planeFragmentShader,
						side: THREE.DoubleSide
				});
				const plane = new THREE.Mesh(planeGeom, planeMat);
				planes.push(plane);
				scene.add(plane);
			}


				// Align planes
				planes[0].rotateY(Math.PI / 2);
				planes[1].rotateX(Math.PI / 2);

				planes[0].material.uniforms.sliceAxis.value = 0;
				planes[1].material.uniforms.sliceAxis.value = 1;
				planes[2].material.uniforms.sliceAxis.value = 2;

				// gui -----------------------------------------------------------------

				function updateMaterial() {
					material.uniforms.threshold.value = parameters.threshold;
					material.uniforms.steps.value = parameters.steps;
					material.uniforms.baseOpacity.value = parameters.baseOpacity;


					planes.forEach(plane => {
						plane.material.uniforms.threshold.value = parameters.threshold;
						plane.material.uniforms.steps.value = parameters.steps;
						plane.material.uniforms.baseOpacity.value = 1.0;
				});
			}

				function updatePlanes() {
					planes.forEach((plane, index) => {
						if (index === 0) {
							plane.visible = planeData['X Plane'];
							plane.position.x = planeData['X Position'];
							plane.material.uniforms.slicePosition.value = planeData['X Position'] + 0.5; // normalize to [0, 1]
						} else if (index === 1) {
							plane.visible = planeData['Y Plane'];
							plane.position.y = planeData['Y Position'];
							plane.material.uniforms.slicePosition.value = planeData['Y Position'] + 0.5;
						} else if (index === 2) {
							plane.visible = planeData['Z Plane'];
							plane.position.z = planeData['Z Position'];
							plane.material.uniforms.slicePosition.value = planeData['Z Position'] + 0.5;
						}
					});
				}

			const parameters = { threshold: 0.5, steps: 200, baseOpacity: 0.5 };

				const gui = new GUI();
				gui.add( parameters, 'threshold', 0, 1, 0.01 ).onChange( updateMaterial );
				gui.add( parameters, 'steps', 0, 300, 1 ).onChange( updateMaterial );
				gui.add(parameters, 'baseOpacity', 0, 1).onChange( updateMaterial );

				const planeFolder = gui.addFolder('Planes');
				const planeData = {
						'X Plane': true,
						'X Position': 0,
						'Y Plane': true,
						'Y Position': 0,
						'Z Plane': true,
						'Z Position': 0,
				};
				

				planeFolder.add(planeData, 'X Plane').name('Show X Plane').onChange(updatePlanes);
				planeFolder.add(planeData, 'X Position', -0.5, 0.5).onChange(updatePlanes);
				planeFolder.add(planeData, 'Y Plane').name('Show Y Plane').onChange(updatePlanes);
				planeFolder.add(planeData, 'Y Position', -0.5, 0.5).onChange(updatePlanes);
				planeFolder.add(planeData, 'Z Plane').name('Show Z Plane').onChange(updatePlanes);
				planeFolder.add(planeData, 'Z Position', -0.5, 0.5).onChange(updatePlanes);

				window.addEventListener( 'resize', onWindowResize );
	}


	function setScene () {

		if ($page.url.pathname == '/') {
			setHome();
		}

	}

	afterNavigate (onNavigate);
	function onNavigate() {

		for( var i = scene.children.length - 1; i >= 0; i--) { 
				let obj = scene.children[i];
				scene.remove(obj); 
		}

		setScene();

	}

	function onWindowResize() {
		let width = window.innerWidth;
		let height = window.innerHeight;

		camera.aspect = width / height;
		camera.updateProjectionMatrix();

		renderer.setSize(width, height);
	}

	function animate() {
		requestAnimationFrame(animate);
		mesh.material.uniforms.cameraPos.value.copy( camera.position );
		renderer.render( scene, camera );
	}
</script>

<div bind:this={container} class:geometry={true} />

<style>
	.geometry {
		position: absolute;
		overflow: hidden;
		z-index: 1;
	}
</style>
