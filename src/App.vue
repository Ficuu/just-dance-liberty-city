<template>
  <div class="pose-container" style="position: relative; width: 640px; height: 480px;">
    <video ref="videoEl" playsinline style="display:none;"></video>
    <canvas ref="canvasEl" width="640" height="480"></canvas>
    <div class="score"
      style="position:absolute;top:10px;left:10px;color:white;font-size:2rem;font-weight:bold;">
      {{ feedback }}
    </div>
    <div class="description"
      style="position:absolute;bottom:10px;left:10px;color:white;font-size:1.2rem;font-weight:bold;text-shadow: 2px 2px 4px #000000;">
      Próxima pose: {{ currentPoseDescription }}
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import { Pose, POSE_CONNECTIONS } from '@mediapipe/pose'
import { Camera } from '@mediapipe/camera_utils'
import { drawConnectors, drawLandmarks } from '@mediapipe/drawing_utils'

const videoEl = ref(null)
const canvasEl = ref(null)

let pose = null
let camera = null

// Coreografía y estado del juego
const choreography = ref([])
const startTime = ref(null)
const feedback = ref('')
const currentPoseDescription = ref('Preparado...')

const currentPoseIndex = ref(0)
const checkPointCounter = ref(0)

// Cargar JSON de la coreografía
async function loadChoreo() {
  const res = await fetch('/choreo.json')
  choreography.value = await res.json()
  // Establecer la primera pose al cargar la coreografía
  if (choreography.value.length > 0) {
    currentPoseDescription.value = choreography.value[currentPoseIndex.value].description
  }
}

// Calcular ángulo entre 3 puntos (A, B, C)
function calculateAngle(A, B, C) {
  const AB = {x: B.x - A.x, y: B.y - A.y}
  const CB = {x: B.x - C.x, y: B.y - C.y}
  const dot = (AB.x * CB.x + AB.y * CB.y)
  const magAB = Math.sqrt(AB.x**2 + AB.y**2)
  const magCB = Math.sqrt(CB.x**2 + CB.y**2)
  const angle = Math.acos(dot / (magAB * magCB))
  return angle * (180 / Math.PI)
}

// Comparar ángulos del usuario con los de la coreografía
function comparePose(userPose, refPose) {
    // Calcular diferencias para todos los ángulos
    const diffLeftArm = Math.abs(userPose.left_arm_angle - refPose.left_arm_angle);
    const diffRightArm = Math.abs(userPose.right_arm_angle - refPose.right_arm_angle);
    const diffLeftShoulder = Math.abs(userPose.left_shoulder_angle - refPose.left_shoulder_angle);
    const diffRightShoulder = Math.abs(userPose.right_shoulder_angle - refPose.right_shoulder_angle);
    
    // Si algún ángulo de referencia es nulo, ignora su diferencia en el promedio
    let totalDiff = 0;
    let count = 0;
    if (refPose.left_arm_angle !== null) { totalDiff += diffLeftArm; count++; }
    if (refPose.right_arm_angle !== null) { totalDiff += diffRightArm; count++; }
    if (refPose.left_shoulder_angle !== null) { totalDiff += diffLeftShoulder; count++; }
    if (refPose.right_shoulder_angle !== null) { totalDiff += diffRightShoulder; count++; }

    const avgDiff = count > 0 ? totalDiff / count : 100;

    if (avgDiff < 15) return 'Perfect';
    if (avgDiff < 30) return 'Good';
    return 'Miss';
}

onMounted(async () => {
  await loadChoreo()

  const canvasCtx = canvasEl.value.getContext('2d')

  pose = new Pose({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`
  })

  pose.setOptions({
    modelComplexity: 1,
    smoothLandmarks: true,
    enableSegmentation: false,
    minDetectionConfidence: 0.5,
    minTrackingConfidence: 0.5
  })

  pose.onResults((results) => {
    canvasCtx.save()
    canvasCtx.clearRect(0, 0, canvasEl.value.width, canvasEl.value.height)
    canvasCtx.drawImage(results.image, 0, 0, canvasEl.value.width, canvasEl.value.height)

    if (results.poseLandmarks) {
      drawConnectors(canvasCtx, results.poseLandmarks, POSE_CONNECTIONS,
        { color: '#00FF00', lineWidth: 4 })
      drawLandmarks(canvasCtx, results.poseLandmarks,
        { color: '#FF0000', lineWidth: 2 })

      const landmarks = results.poseLandmarks
      
      // Calcular ángulos de los brazos y hombros del usuario
      const userPose = {
        left_arm_angle: calculateAngle(landmarks[11], landmarks[13], landmarks[15]),
        right_arm_angle: calculateAngle(landmarks[12], landmarks[14], landmarks[16]),
        left_shoulder_angle: calculateAngle(landmarks[13], landmarks[11], landmarks[23]),
        right_shoulder_angle: calculateAngle(landmarks[14], landmarks[12], landmarks[24])
      }

      // Lógica de avance de la coreografía
      if (currentPoseIndex.value < choreography.value.length) {
        const refPose = choreography.value[currentPoseIndex.value]
        
        const poseStatus = comparePose(userPose, refPose.pose)
        feedback.value = poseStatus
        
        if (poseStatus === 'Perfect' || poseStatus === 'Good') {
          checkPointCounter.value++
          if (checkPointCounter.value > 30) { // 30 frames ~ 1 segundo a 30fps
            currentPoseIndex.value++
            checkPointCounter.value = 0
            
            if (currentPoseIndex.value < choreography.value.length) {
              currentPoseDescription.value = choreography.value[currentPoseIndex.value].description
            } else {
              currentPoseDescription.value = '¡Coreografía terminada!'
              feedback.value = '¡Excelente!'
            }
          }
        } else {
          checkPointCounter.value = 0
        }
      }
    }

    canvasCtx.restore()
  })

  // Iniciar cámara
  camera = new Camera(videoEl.value, {
    onFrame: async () => {
      await pose.send({ image: videoEl.value })
    },
    width: 640,
    height: 480
  })
  camera.start()
})

onBeforeUnmount(() => {
  if (camera) camera.stop()
})
</script>

<style scoped>
.pose-container {
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>