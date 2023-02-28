<script>
import { Renderer, Program, Texture, Mesh, Vec2, Vec4, Flowmap, Geometry } from 'ogl'
import pluto from '/src/assets/pluto.png'

// Shaders

const vertex = `
    attribute vec2 uv;
    attribute vec2 position;
    varying vec2 vUv;
    void main() {
        vUv = uv;
        gl_Position = vec4(position, 0, 1);
    }
`;

const fragment = `
    precision highp float;
    precision highp int;
    uniform sampler2D tWater;
    uniform sampler2D tFlow;
    uniform float uTime;
    varying vec2 vUv;
    uniform vec4 res;

    void main() {

        // R and G values are velocity in the x and y direction
        // B value is the velocity length

        vec3 flow = texture2D(tFlow, vUv).rgb;
        vec2 uv = .5 * gl_FragCoord.xy / res.xy ;
        vec2 myUV = (uv - vec2(0.5))*res.zw + vec2(0.5);
        myUV -= flow.xy * (0.15 * 0.5);

        gl_FragColor = texture2D(tWater, myUV);
    }
`;

export default {

    mounted() {

        const el = this.$refs.lbg

        // Renderer & append

        const renderer = new Renderer({dpr: 2});
        const gl = renderer.gl;
        el.appendChild(gl.canvas);

        // Touch

        const isTouchCapable = "ontouchstart" in window;

        // Init resize

        var imgSize = [1512, 982];
        var imageAspect = imgSize[1] / imgSize[0];
        var a1, a2;
        var aspect = 1;

        function setSize(){
            if (window.innerHeight / window.innerWidth < imageAspect) {
                a1 = 1;
                a2 = window.innerHeight / window.innerWidth / imageAspect;
            } else {
                a1 = (window.innerWidth / window.innerHeight) * imageAspect;
                a2 = 1;
            }
        }
        setSize();

        function resize() {

            setSize();

            mesh.program.uniforms.res.value = new Vec4(
                window.innerWidth,
                window.innerHeight,
                a1,
                a2
            );

            renderer.setSize(window.innerWidth, window.innerHeight);
            aspect = window.innerWidth / window.innerHeight;
        }

        // Flowmap

        const flowmap = new Flowmap(gl, { falloff: 0.4, dissipation: 0.9999 });

        // Geometry

        const geometry = new Geometry(gl, {
            position: {
                size: 2,
                data: new Float32Array([-1, -1, 3, -1, -1, 3])
            },
            uv: { size: 2, data: new Float32Array([0, 0, 2, 0, 0, 2]) }
        });

        // Texture

        const texture = new Texture(gl);
        const bG = new Image();
        bG.onload = () => (texture.image = bG);
        bG.src = pluto

        // Program

        const program = new Program(gl, {
            vertex,
            fragment,
            uniforms: {
                uTime: { value: 0 },
                tWater: { value: texture },
                res: {
                    value: new Vec4(window.innerWidth, window.innerHeight, a1, a2)
                },
                img: { value: new Vec2(imgSize[0], imgSize[1]) },
                tFlow: flowmap.uniform
            }
        });

        // Mesh

        const mesh = new Mesh(gl, { geometry, program });

        // Call resize

        window.addEventListener("resize", resize, false);
        resize();

        // Interactions

        if (isTouchCapable) {
            window.addEventListener("touchstart", updateMouse, false);
            window.addEventListener("touchmove", updateMouse, { passive: false });
        } else {
            window.addEventListener("mousemove", updateMouse, false);
        }
        
        const mouse = new Vec2(-1);
        const velocity = new Vec2();
        let lastTime;
        const lastMouse = new Vec2();

        function updateMouse(e) {
            e.preventDefault();
            
            if (e.changedTouches && e.changedTouches.length) {
                e.x = e.changedTouches[0].pageX;
                e.y = e.changedTouches[0].pageY;
            }
            if (e.x === undefined) {
                e.x = e.pageX;
                e.y = e.pageY;
            }

            // Get mouse value in 0 to 1 range, with y flipped
            mouse.set(e.x / gl.renderer.width, 1.0 - e.y / gl.renderer.height);

            // Calculate velocity
            if (!lastTime) {
                // First frame
                lastTime = performance.now();
                lastMouse.set(e.x, e.y);
            }
            const deltaX = e.x - lastMouse.x;
            const deltaY = e.y - lastMouse.y;
            lastMouse.set(e.x, e.y);
            let time = performance.now();

            // Avoid dividing by 0
            let delta = Math.max(10.4, time - lastTime);
            lastTime = time;
            velocity.x = deltaX / delta;
            velocity.y = deltaY / delta;
            
            // Flag update to prevent hanging velocity values when not moving
            velocity.needsUpdate = true;
        }

        requestAnimationFrame(update);
        function update(t) {
            requestAnimationFrame(update);
            // Reset velocity when mouse not moving
            if (!velocity.needsUpdate) {
                mouse.set(-1);
                velocity.set(0);
            }
            velocity.needsUpdate = false;
            // Update flowmap inputs
            flowmap.aspect = aspect;
            flowmap.mouse.copy(mouse);
            // Ease velocity input, slower when fading out
            flowmap.velocity.lerp(velocity, velocity.len ? 0.15 : 0.1);
            flowmap.update();
            program.uniforms.uTime.value = t * 0.01;
            renderer.render({ scene: mesh });
        }
    }
}
</script>

<template>
    <div class="liquid-background" ref="lbg"></div>
</template>

<style scoped>
.liquid-background {
    position: fixed;
    top: 0;
    left: 0;
    z-index: -2;
}
</style>