<script lang="ts">
	import { onMount } from "svelte";
	import SectionDetailModal from "./SectionDetailModal.svelte";
	let dayLabels: any[][] = [[], [], [], [], []];
	let daysCodes = ["M", "Tu", "W", "Th", "F"];
	let dayNames = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"];

	export let colorMap: Map<string, string> = new Map<string, string>();
	export let scheduleData: any;
	export let scheduleIndex: number = 0;

	const hourHeight = 60; // Height in pixels for one hour
	const dayHeaderHeight = 50; // Height of day header
	let earliestStart = 480; // 8:00 AM in minutes from midnight
	let latestEnd = 1200; // 8:00 PM in minutes from midnight
	const totalHours = (latestEnd - earliestStart) / 60;
	const scheduleHeight = totalHours * hourHeight;

	// Generate time labels for the schedule with more precision
	let timeLabels: string[] = [];
	for (let i = earliestStart; i <= latestEnd; i += 60) {
		const hour = Math.floor(i / 60);
		const meridiem = hour >= 12 ? "PM" : "AM";
		const displayHour = hour > 12 ? hour - 12 : hour === 0 ? 12 : hour;
		timeLabels.push(`${displayHour}:00 ${meridiem}`);
	}

	let showSectionModal = false;
	let selectedSectionId = "";
	let selectedCourseId = "";

	// Function to open section details
	function openSectionDetails(courseId: string, sectionId: string) {
		selectedCourseId = courseId;
		selectedSectionId = sectionId;
		showSectionModal = true;
	}

	// Function to close section modal
	function closeSectionModal() {
		showSectionModal = false;
		// Don't reset the IDs until after animation completes
		setTimeout(() => {
			selectedSectionId = "";
			selectedCourseId = "";
		}, 300);
	}

	onMount(() => {
		// Process schedule data
		Object.keys(scheduleData).forEach((element) => {
			const meetings = scheduleData[element]?.meetings;
			if (meetings) {
				meetings.forEach((classTime: any) => {
					const days = classTime.days.toString();
					const startDate = createDate(classTime.start_time);
					const endDate = createDate(classTime.end_time);

					const { startPixels, heightPixels } = calculateSlotTimes(
						startDate,
						endDate,
					);

					daysCodes.forEach((dayCode, k) => {
						if (days.includes(dayCode)) {
							const slot = {
								sectionCode: scheduleData[element]["number"],
								days: dayCode,
								startPixels, // pixels
								startTime: getFormattedTime(startDate),
								endTime: getFormattedTime(endDate),
								class: element,
								professor:
									scheduleData[element][
										"instructors"
									].toString(),
								location: `${classTime.building} ${classTime.room}`,
								prof_rating:
									scheduleData[element]["prof_weight"],
								heightPixels, // pixels
								// Add raw time values for overlap detection
								rawStartTime:
									startDate.getHours() * 60 +
									startDate.getMinutes(),
								rawEndTime:
									endDate.getHours() * 60 +
									endDate.getMinutes(),
							};
							dayLabels[k].push(slot);
						}
					});
				});
			}
		});

		// Find overlapping slots and adjust their horizontal position
		for (let dayIndex = 0; dayIndex < dayLabels.length; dayIndex++) {
			const dayClasses = dayLabels[dayIndex];

			if (dayClasses.length > 1) {
				// Sort by start time
				dayClasses.sort((a, b) => a.rawStartTime - b.rawStartTime);

				// Track overlapping groups
				const overlapGroups: any[][] = [];
				let currentGroup: any[] = [];

				// Group overlapping classes
				for (let i = 0; i < dayClasses.length; i++) {
					const current = dayClasses[i];

					if (i === 0) {
						// Start the first group with the first class
						currentGroup = [current];
					} else {
						// Check if this class overlaps with any in the current group
						const overlaps = currentGroup.some(
							(prev) =>
								current.rawStartTime < prev.rawEndTime &&
								current.rawEndTime > prev.rawStartTime,
						);

						if (overlaps) {
							// Add to current group if overlapping
							currentGroup.push(current);
						} else {
							// Start a new group if no overlap
							if (currentGroup.length > 0) {
								overlapGroups.push([...currentGroup]);
							}
							currentGroup = [current];
						}
					}
				}

				// Add the last group if not empty
				if (currentGroup.length > 0) {
					overlapGroups.push(currentGroup);
				}

				// Assign horizontal position for overlapping classes
				overlapGroups.forEach((group) => {
					if (group.length > 1) {
						for (let i = 0; i < group.length; i++) {
							group[i].horizontalPosition = i;
							group[i].totalOverlap = group.length;
						}
					}
				});
			}
		}

		// Trigger reactivity by reassigning dayLabels
		dayLabels = [...dayLabels];
	});

	const createDate = (dateStr: string) => {
		// Fix time parsing to be more robust
		const trimmedStr = dateStr.trim().toLowerCase();
		const isPM = trimmedStr.includes("pm");

		// Remove am/pm and extract hours and minutes
		const timeOnly = trimmedStr.replace(/am|pm/i, "").trim();
		const [hoursStr, minutesStr] = timeOnly.split(":");

		let hours = parseInt(hoursStr);
		const minutes = parseInt(minutesStr);

		// Proper 12-hour to 24-hour conversion
		if (isPM && hours < 12) {
			hours += 12;
		} else if (!isPM && hours === 12) {
			hours = 0;
		}

		const date = new Date();
		date.setHours(hours);
		date.setMinutes(minutes);
		date.setSeconds(0);
		return date;
	};

	const getFormattedTime = (date: Date) => {
		return new Intl.DateTimeFormat("en-US", {
			hour: "numeric",
			minute: "2-digit",
			hour12: true,
		}).format(date);
	};

	// Improved slot time calculation for precise alignment with the grid
	const calculateSlotTimes = (startDate: Date, endDate: Date) => {
		const startMinutes = startDate.getHours() * 60 + startDate.getMinutes();
		const endMinutes = endDate.getHours() * 60 + endDate.getMinutes();

		// Calculate position in pixels directly (no percentages)
		const pixelsPerMinute = hourHeight / 60;
		const startPixels = (startMinutes - earliestStart) * pixelsPerMinute;
		const heightPixels = (endMinutes - startMinutes) * pixelsPerMinute;

		return { startPixels, heightPixels };
	};

	function formatSlots(day: any[]) {
		// Change sort from "a.start" to "a.startPixels"
		return day.sort((a, b) => a.startPixels - b.startPixels);
	}

	// Function to calculate average professor rating
	function calculateAverageRating(): string {
		if (!scheduleData) return "N/A";

		let totalRating = 0;
		let countRated = 0;

		Object.keys(scheduleData).forEach((className) => {
			if (
				scheduleData[className] &&
				scheduleData[className]["prof_weight"]
			) {
				totalRating += parseFloat(
					scheduleData[className]["prof_weight"],
				);
				countRated++;
			}
		});

		if (countRated === 0) return "N/A";
		return (totalRating / countRated).toFixed(2);
	}

	// Calculate the average rating when the component initializes
	const avgProfRating = calculateAverageRating();
</script>

<div class="schedule-wrapper">
	<div class="schedule-header">
		<div class="schedule-info">
			<div class="schedule-number">
				<span class="schedule-badge">#{scheduleIndex + 1}</span>
				<h3>Schedule</h3>
			</div>
			<div class="rating-badge">
				<svg
					xmlns="http://www.w3.org/2000/svg"
					width="16"
					height="16"
					viewBox="0 0 24 24"
					fill="currentColor"
					stroke="none"
				>
					<polygon
						points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"
					></polygon>
				</svg>
				<span class="rating-value">{avgProfRating}</span>
				<span class="rating-label">Avg. Professor Rating</span>
			</div>
		</div>
	</div>

	<div class="schedule-container">
		<!-- Time labels column with fixed dimensions -->
		<div class="time-labels" style="padding-top: {dayHeaderHeight}px;">
			{#each timeLabels as time, i}
				<div class="time-label" style="height: {hourHeight}px;">
					{time}
				</div>
			{/each}
		</div>

		<!-- Content area with days and schedule -->
		<div class="schedule-content">
			<!-- Day headers with fixed height -->
			<div class="day-headers" style="height: {dayHeaderHeight}px;">
				{#each dayNames as day, i}
					<div class="day-header">{day}</div>
				{/each}
			</div>

			<!-- Scrollable grid area with fixed height -->
			<div class="grid-area" style="height: {scheduleHeight}px;">
				<!-- Background grid lines -->
				<div class="grid-lines">
					{#each timeLabels as _, i}
						<div
							class="grid-line"
							style="top: {i *
								hourHeight}px; height: {hourHeight}px;"
						></div>
					{/each}
				</div>

				<!-- Day columns with events -->
				<div class="day-columns">
					{#each daysCodes as day, i}
						<div class="schedule-column">
							{#each formatSlots(dayLabels[i] || []) as slot}
								<!-- Schedule slot with absolute positioning -->
								<!-- svelte-ignore a11y-click-events-have-key-events -->
								<!-- svelte-ignore a11y-no-static-element-interactions -->
								<div
									class="schedule-slot"
									style="
										top: {slot.startPixels}px; 
										height: {slot.heightPixels}px; 
										background: {colorMap.get(slot.class) || '#f0f0f0'};
										{slot.horizontalPosition !== undefined
										? `left: calc(${slot.horizontalPosition * (100 / slot.totalOverlap)}%); 
												width: calc(${100 / slot.totalOverlap}%);`
										: 'left: 0; right: 0;'}
									"
									on:click={() =>
										openSectionDetails(
											slot.class,
											`${slot.class}-${slot.sectionCode}`,
										)}
								>
									<div class="slot-content">
										<!-- Consistent layout for all slots -->
										<div class="class-row">
											<span class="class-name"
												>{slot.class}</span
											>
											<span class="section-number"
												>§{slot.sectionCode}</span
											>
										</div>
										<div class="info-row">
											<span class="slot-location"
												>{slot.location}</span
											>
											<span class="slot-time"
												>{slot.startTime.replace(
													":00",
													"",
												)}-{slot.endTime.replace(
													":00",
													"",
												)}</span
											>
										</div>
									</div>

									<!-- Show tooltip on hover for all slots, regardless of size -->
									<div
										class="slot-tooltip"
										style="
											background: {colorMap.get(slot.class) || '#f0f0f0'};
											border-color: {colorMap.get(slot.class)
											? 'rgba(0, 0, 0, 0.1)'
											: 'rgba(0, 0, 0, 0.1)'};
										"
									>
										<div class="tooltip-professor">
											<svg
												xmlns="http://www.w3.org/2000/svg"
												width="12"
												height="12"
												viewBox="0 0 24 24"
												fill="none"
												stroke="currentColor"
												stroke-width="2"
											>
												<path
													d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"
												></path>
												<circle cx="12" cy="7" r="4"
												></circle>
											</svg>
											{slot.professor}
											{#if slot.prof_rating}
												<span class="tooltip-rating"
													>({Number(
														slot.prof_rating,
													).toFixed(1)} ⭐)</span
												>
											{/if}
										</div>
									</div>
								</div>
							{/each}
						</div>
					{/each}
				</div>
			</div>
		</div>
	</div>
</div>

{#if showSectionModal}
	<SectionDetailModal
		showModal={showSectionModal}
		sectionId={selectedSectionId}
		courseId={selectedCourseId}
		on:close={closeSectionModal}
	/>
{/if}

<style>
	.schedule-wrapper {
		width: 100%;
		margin: 0 0 2rem 0;
		overflow-x: auto; /* Allow horizontal scroll only within this component if needed */
	}

	/* Container uses CSS Grid for precise alignment */
	.schedule-container {
		display: grid;
		grid-template-columns: 80px 1fr;
		grid-template-rows: auto;
		width: 100%;
		min-width: 600px; /* Minimum width to prevent squishing */
		max-width: 100%;
		margin: 0 0 40px 0;
		border-radius: 8px;
		overflow: hidden;
		box-shadow: var(--shadow-sm);
		box-sizing: border-box;
		border-top-left-radius: 0;
		border-top-right-radius: 0;
		background-color: white;
	}

	/* Time labels column with exact alignment */
	.time-labels {
		position: relative;
		display: flex;
		flex-direction: column;
		background-color: #f9f9f9;
		border-right: 1px solid #eaeaea;
		z-index: 2;
		box-sizing: border-box;
		padding: 0; /* Remove padding */
	}

	.time-label {
		display: flex;
		align-items: flex-start; /* Align to top */
		justify-content: center;
		font-size: 0.8rem;
		color: #666;
		border-bottom: 1px solid #eaeaea;
		box-sizing: border-box;
		position: relative;
		text-align: center;
		padding-top: 4px; /* Small top padding */
	}

	/* Schedule content area containing days and grid */
	.schedule-content {
		display: flex;
		flex-direction: column;
		position: relative;
		width: 100%;
	}

	/* Day headers with fixed height */
	.day-headers {
		display: grid;
		grid-template-columns: repeat(5, 1fr);
		background-color: #e21833;
		color: white;
		z-index: 2;
	}

	.day-header {
		display: flex;
		align-items: center;
		justify-content: center;
		font-weight: bold;
		border-right: 1px solid rgba(255, 255, 255, 0.2);
	}

	/* Grid area containing both lines and events */
	.grid-area {
		position: relative; /* Ensure content is laid out properly */
		width: 100%;
		background-color: white;
	}

	/* Grid lines with exact height matching time labels - FIXED */
	.grid-lines {
		position: absolute;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		z-index: 0; /* Send these behind day-columns */
		pointer-events: none;
	}

	.grid-line {
		position: absolute; /* Absolute positioning for precise placement */
		width: 100%;
		border-bottom: 1px solid #eaeaea;
		box-sizing: border-box;
		height: 1px; /* Visual height of line is 1px */
		left: 0;
		right: 0;
	}

	/* Day columns container uses CSS Grid for equal column widths */
	.day-columns {
		display: grid;
		grid-template-columns: repeat(5, 1fr);
		position: absolute; /* Position absolutely to align with grid lines */
		top: 0;
		width: 100%;
		height: 100%;
		z-index: 1; /* Above grid lines, below event slots */
	}

	.schedule-column {
		position: relative;
		border-right: 1px solid #eaeaea;
		height: 100%;
	}

	/* Ensure slots are positioned precisely */
	.schedule-slot {
		position: absolute;
		border-radius: 6px;
		color: #333;
		padding: 4px;
		box-sizing: border-box;
		overflow: visible; /* Changed from hidden to allow tooltip to extend outside */
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
		transition: all 0.25s ease;
		z-index: 2; /* Above grid lines and day columns */
		cursor: pointer;
	}

	.schedule-slot:hover {
		transform: scale(1.02);
		box-shadow: 0 5px 10px rgba(0, 0, 0, 0.15);
		z-index: 10;
	}

	/* Improved slot content layout */
	.slot-content {
		height: 100%;
		width: 100%;
		padding: 6px;
		display: flex;
		flex-direction: column;
		justify-content: space-between; /* Space items evenly in the container */
		overflow: hidden;
		position: relative; /* For absolute positioning of hover elements */
	}

	.class-row {
		display: flex;
		justify-content: space-between;
		align-items: center;
		width: 100%;
		overflow: hidden;
	}

	.info-row {
		display: flex;
		justify-content: flex-start; /* Left align info */
		align-items: center;
		width: 100%;
		overflow: hidden;
		font-size: 0.75rem;
		line-height: 1.2;
	}

	/* Class name is the only bold element */
	.class-name {
		font-weight: bold;
		font-size: 0.85rem;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		flex: 1;
	}

	/* Section number now positioned inline */
	.section-number {
		font-size: 0.75rem;
		opacity: 0.8;
		font-weight: normal;
		white-space: nowrap;
		padding-left: 4px;
		flex-shrink: 0;
	}

	/* All other text is consistent weight and size */
	.slot-location,
	.slot-time {
		font-weight: normal;
		font-size: 0.75rem;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	/* Tooltip redesigned to appear as an extension of the slot */
	.slot-tooltip {
		display: none;
		position: absolute;
		top: calc(100% - 1px); /* Overlap slightly to connect with slot */
		left: -1px; /* Align with parent border */
		right: -1px;
		border-radius: 0 0 6px 6px; /* Round only bottom corners */
		padding: 8px;
		box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
		z-index: 1;
		font-size: 0.85rem;
		line-height: 1.4;
		color: #333;
		border: 1px solid rgba(0, 0, 0, 0.1);
		border-top: none; /* Remove top border for seamless connection */
		transform-origin: top center;
		transition:
			transform 0.2s,
			opacity 0.2s;
		opacity: 0;
		transform: scaleY(0.8);
		pointer-events: none; /* Don't interfere with slot click events */
	}

	.schedule-slot:hover .slot-tooltip {
		display: block;
		opacity: 1;
		transform: scaleY(1);
		pointer-events: auto; /* Re-enable pointer events when visible */
	}

	/* For slots near the bottom of the schedule, show tooltip above */
	.schedule-column:nth-last-child(-n + 2)
		.schedule-slot:nth-last-child(-n + 2)
		.slot-tooltip {
		top: auto;
		bottom: calc(100% - 1px);
		border-radius: 6px 6px 0 0; /* Round top corners instead */
		border-bottom: none;
		border-top: 1px solid rgba(0, 0, 0, 0.1);
		transform-origin: bottom center;
	}

	/* Tooltip content styling - simplified to remove lines */
	.tooltip-professor {
		display: flex;
		align-items: center;
		gap: 6px;
		white-space: normal; /* Allow wrapping for long professor names */
		line-height: 1.3;
		width: 100%;
	}

	.tooltip-rating {
		margin-left: 4px;
		font-size: 0.8rem;
	}

	.class-row {
		display: flex;
		justify-content: space-between;
		align-items: center;
		width: 100%;
		overflow: hidden;
	}

	.info-row {
		display: flex;
		justify-content: space-between;
		align-items: center;
		width: 100%;
		overflow: hidden;
		font-size: 0.75rem;
	}

	/* Class name is the only bold element */
	.class-name {
		font-weight: bold;
		font-size: 0.8rem;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		flex: 1;
	}

	/* Section number now positioned inline */
	.section-number {
		font-size: 0.7rem;
		opacity: 0.9;
		font-weight: normal;
		white-space: nowrap;
		padding-left: 4px;
		flex-shrink: 0;
	}

	/* All other text is consistent weight and size */
	.slot-location,
	.slot-time {
		font-weight: normal;
		font-size: 0.75rem;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.slot-location {
		flex: 1;
		text-align: left;
	}

	.slot-time {
		text-align: right;
		flex-shrink: 0;
		margin-left: 4px;
	}

	/* Make the schedule more responsive */
	@media (max-width: 768px) {
		.schedule-container {
			display: flex;
			flex-direction: column;
			width: 100%;
			overflow-x: auto; /* Allow horizontal scroll on narrow screens */
		}

		.time-labels {
			display: none; /* Hide on small screens to save space */
		}

		.schedule-content {
			min-width: 600px; /* Minimum width to ensure readability */
		}

		.schedule-wrapper {
			margin: 0 -16px; /* Negative margin to allow schedule to break out of container */
			padding: 0 16px; /* Add padding to compensate and maintain visual alignment */
			width: calc(100% + 32px); /* Compensate for negative margins */
		}

		/* ...other mobile styles... */
	}

	/* Restore original schedule header styling */
	.schedule-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: var(--space-3) var(--space-4);
		background-color: white;
		border-top-left-radius: var(--radius-lg);
		border-top-right-radius: var(--radius-lg);
		border-bottom: 2px solid var(--primary);
		margin-bottom: -1px; /* Connect with schedule container */
	}

	.schedule-info {
		display: flex;
		justify-content: space-between;
		width: 100%;
		align-items: center;
	}

	.schedule-number {
		display: flex;
		align-items: center;
		gap: var(--space-2);
	}

	.schedule-number h3 {
		margin: 0;
		font-size: 1.125rem;
		font-weight: 600;
		color: var(--neutral-800);
	}

	.schedule-badge {
		background-color: var(--primary);
		color: white;
		border-radius: var(--radius-full);
		padding: var(--space-1) var(--space-2);
		font-size: 0.875rem;
		font-weight: 600;
		min-width: 24px;
		text-align: center;
	}

	.rating-badge {
		display: flex;
		align-items: center;
		gap: var(--space-2);
		background-color: white;
		padding: var(--space-2) var(--space-3);
		border-radius: var(--radius-full);
		box-shadow: var(--shadow-sm);
		color: var(--neutral-600);
	}

	.rating-badge svg {
		color: #ffc107;
	}

	.rating-value {
		color: var(--neutral-900);
		font-weight: 600;
		font-size: 0.875rem;
	}

	.rating-label {
		color: var(--neutral-600);
		font-size: 0.75rem;
	}
</style>
