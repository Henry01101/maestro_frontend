<template>
    <div class="formulario">
        <div class="header">
            <h2>{{ currentSection.title }}</h2>
            <p class="progress">Sección {{ currentIndex + 1 }} de {{ sectionsInternal.length }}</p>
        </div>

        <div class="questions">
            <div
                v-for="q in currentSection.questions"
                :key="q.id"
                class="question"
            >
                <label :for="q.id" class="q-label">
                    {{ q.label }} <span v-if="q.required" class="req">*</span>
                </label>

                <!-- completacion (texto) -->
                <textarea
                    v-if="q.type === 'text'"
                    :id="q.id"
                    v-model="answers[q.id]"
                    :placeholder="q.placeholder || ''"
                    class="input"
                ></textarea>

                <!-- select (elección única) -->
                <select
                    v-else-if="q.type === 'select'"
                    :id="q.id"
                    v-model="answers[q.id]"
                    class="input"
                >
                    <option value="" disabled hidden>Seleccione...</option>
                    <option
                        v-for="opt in q.options || []"
                        :key="opt.value"
                        :value="opt.value"
                    >{{ opt.label }}</option>
                </select>

                <!-- checkbox (elección múltiple) -->
                <div v-else-if="q.type === 'checkbox'" class="checkbox-group">
                    <label
                        v-for="opt in q.options || []"
                        :key="opt.value"
                        class="checkbox-label"
                    >
                        <input
                            type="checkbox"
                            :value="opt.value"
                            :checked="answers[q.id] && answers[q.id].includes(opt.value)"
                            @change="onCheckboxChange(q.id, opt.value, $event.target.checked)"
                        />
                        {{ opt.label }}
                    </label>
                </div>

                <div v-if="errors[q.id]" class="error">{{ errors[q.id] }}</div>
            </div>
        </div>

        <div class="actions">
            <button @click="prevSection" :disabled="currentIndex === 0">Anterior</button>
            <button v-if="!isLastSection" @click="nextSection">Siguiente</button>
            <button v-else @click="finish">Finalizar</button>
        </div>
    </div>
</template>

<script>
import { defineComponent, reactive, computed, toRefs, watch } from 'vue';

export default defineComponent({
    name: 'Formulario',
    props: {
        // Se puede pasar desde el padre. Si no, se usa un ejemplo por defecto.
        sections: {
            type: Array,
            default: () => ([
                {
                    id: 's1',
                    title: 'Información personal',
                    questions: [
                        { id: 'q1', type: 'text', label: 'Nombre completo', placeholder: 'Escribe tu nombre', required: true },
                        { id: 'q2', type: 'select', label: 'País', options: [
                            { value: 'es', label: 'España' },
                            { value: 'mx', label: 'México' },
                            { value: 'ar', label: 'Argentina' }
                        ], required: true },
                    ]
                },
                {
                    id: 's2',
                    title: 'Preferencias',
                    questions: [
                        { id: 'q3', type: 'checkbox', label: 'Tecnologías que conoces', options: [
                            { value: 'vue', label: 'Vue' },
                            { value: 'react', label: 'React' },
                            { value: 'angular', label: 'Angular' }
                        ], required: true },
                        { id: 'q4', type: 'text', label: 'Comentarios adicionales', placeholder: 'Opcional', required: false },
                    ]
                }
            ])
        }
    },
    emits: ['submit'],
    setup(props, { emit }) {
        const state = reactive({
            sectionsInternal: props.sections.slice(),
            currentIndex: 0,
            answers: {}, // id -> value
            errors: {}
        });

        // Inicializar respuestas según preguntas
        const initAnswers = () => {
            state.answers = {};
            state.errors = {};
            for (const sec of state.sectionsInternal) {
                for (const q of sec.questions) {
                    if (q.type === 'checkbox') state.answers[q.id] = [];
                    else state.answers[q.id] = '';
                    state.errors[q.id] = '';
                }
            }
        };

        initAnswers();

        // Si las secciones cambian desde el padre, reiniciar
        watch(
            () => props.sections,
            (newSecs) => {
                state.sectionsInternal = newSecs.slice();
                initAnswers();
                state.currentIndex = 0;
            },
            { deep: true }
        );

        const currentSection = computed(() => state.sectionsInternal[state.currentIndex] || { title: '', questions: [] });
        const currentIndex = computed(() => state.currentIndex);
        const isLastSection = computed(() => state.currentIndex === state.sectionsInternal.length - 1);

        const validateSection = () => {
            let valid = true;
            const sec = currentSection.value;
            for (const q of sec.questions) {
                state.errors[q.id] = '';
                if (q.required) {
                    const val = state.answers[q.id];
                    if (q.type === 'checkbox') {
                        if (!Array.isArray(val) || val.length === 0) {
                            state.errors[q.id] = 'Selecciona al menos una opción';
                            valid = false;
                        }
                    } else {
                        if (!val || String(val).trim() === '') {
                            state.errors[q.id] = 'Este campo es obligatorio';
                            valid = false;
                        }
                    }
                }
            }
            return valid;
        };

        const nextSection = () => {
            if (!validateSection()) return;
            if (state.currentIndex < state.sectionsInternal.length - 1) state.currentIndex++;
        };

        const prevSection = () => {
            if (state.currentIndex > 0) state.currentIndex--;
        };

        const finish = () => {
            // validar toda la sección final antes de enviar
            if (!validateSection()) return;
            // Opcional: validar todo el formulario (todas las secciones)
            const allErrors = {};
            let ok = true;
            for (const sec of state.sectionsInternal) {
                for (const q of sec.questions) {
                    allErrors[q.id] = '';
                    if (q.required) {
                        const val = state.answers[q.id];
                        if (q.type === 'checkbox') {
                            if (!Array.isArray(val) || val.length === 0) {
                                allErrors[q.id] = 'Selecciona al menos una opción';
                                ok = false;
                            }
                        } else {
                            if (!val || String(val).trim() === '') {
                                allErrors[q.id] = 'Este campo es obligatorio';
                                ok = false;
                            }
                        }
                    }
                }
            }
            state.errors = Object.assign({}, state.errors, allErrors);
            if (!ok) return;

            // Emitir respuestas
            emit('submit', JSON.parse(JSON.stringify(state.answers)));
        };

        const onCheckboxChange = (qid, value, checked) => {
            const arr = state.answers[qid] || [];
            if (checked) {
                if (!arr.includes(value)) arr.push(value);
            } else {
                const idx = arr.indexOf(value);
                if (idx !== -1) arr.splice(idx, 1);
            }
            // force update reference for reactivity (in case)
            state.answers[qid] = Array.from(arr);
            state.errors[qid] = '';
        };

        return {
            ...toRefs(state),
            currentSection,
            currentIndex,
            isLastSection,
            nextSection,
            prevSection,
            finish,
            onCheckboxChange
        };
    }
});
</script>

<style scoped>
.formulario {
    max-width: 700px;
    margin: 0 auto;
    padding: 16px;
    border: 1px solid #e6e6e6;
    border-radius: 8px;
    background: #fff;
    font-family: Arial, sans-serif;
}
.header h2 {
    margin: 0 0 8px;
}
.progress {
    color: #666;
    font-size: 0.9rem;
    margin-bottom: 12px;
}
.question {
    margin-bottom: 14px;
}
.q-label {
    display: block;
    margin-bottom: 6px;
    font-weight: 600;
}
.req {
    color: #d00;
    margin-left: 4px;
}
.input {
    width: 100%;
    min-height: 40px;
    padding: 8px;
    border: 1px solid #cfcfcf;
    border-radius: 4px;
    box-sizing: border-box;
    font-size: 0.95rem;
}
textarea.input {
    min-height: 80px;
}
.checkbox-group {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
}
.checkbox-label {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.95rem;
}
.actions {
    margin-top: 18px;
    display: flex;
    gap: 8px;
}
button {
    padding: 8px 14px;
    border: none;
    background: #2b6cb0;
    color: white;
    border-radius: 4px;
    cursor: pointer;
}
button[disabled] {
    opacity: 0.5;
    cursor: default;
}
button:hover:not([disabled]) {
    background: #235a93;
}
.error {
    color: #b00020;
    font-size: 0.85rem;
    margin-top: 6px;
}
</style>